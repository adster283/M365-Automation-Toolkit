# Collect Device Crash Logs

Use this script when investigating blue screens, random restarts, unexpected shutdowns, freezes, driver crashes, or sleep and wake problems.

For the investigation workflow, see [README.md](./README.md).

> **Important:** Run this as administrator. Crash dumps can contain sensitive data and can make the output ZIP very large.

## Output

```text
C:\Temp\CrashLogs
```

## Steps
1. Launch Admin CMD window
2. Copy and paste the below script
3. Let the script run until it says it is complete
4. Copy the "C:\Temp\CrashLogs" folder to your laptop for investigation.

The script automatically closes its elevated Command Prompt on success or failure.

## Script

```
@echo off
setlocal EnableExtensions EnableDelayedExpansion

rem ============================================================
rem Collect Device Crash Logs - Hardened / Verbose Edition v7
rem ============================================================

set "OUTPUT=C:\Temp\CrashLogs"
set "ROBOLOG=%OUTPUT%\Robocopy.log"
set "ERRORLOG=%OUTPUT%\Collection-Errors.txt"
set "SUMMARY=%OUTPUT%\Collection-Summary.txt"

set /a OKCOUNT=0
set /a WARNCOUNT=0
set /a SKIPCOUNT=0
set /a FAILCOUNT=0
set "DUMPFOUND=0"

rem ------------------------------------------------------------
rem Administrator check
rem ------------------------------------------------------------

echo Checking administrator rights...
fltmc >nul 2>&1

if errorlevel 1 (
    powershell.exe -NoProfile -Command "Write-Host '[FAIL] This script must be run as administrator.' -ForegroundColor Red" 2>nul
    start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('This script must be run as administrator. Right-click it and select Run as administrator.','Administrator rights required','OK','Error')" 2>nul
    exit /b 1
)

rem ------------------------------------------------------------
rem Clean and recreate working area
rem ------------------------------------------------------------

if not exist "C:\Temp" (
    mkdir "C:\Temp" >nul 2>&1
)

if not exist "C:\Temp" (
    echo ERROR: Could not create C:\Temp
    exit /b 1
)

if exist "%OUTPUT%" (
    rmdir /s /q "%OUTPUT%" >nul 2>&1
)

if exist "%OUTPUT%" (
    powershell.exe -NoProfile -Command "Write-Host '[FAIL] Could not remove the previous C:\Temp\CrashLogs folder. A file may be locked.' -ForegroundColor Red" 2>nul
    exit /b 1
)

rem Remove any archive left by an older version or a previous manual run.
rem This script does not create ZIP files itself.
if exist "C:\Temp\CrashLogs.zip" (
    del /f /q "C:\Temp\CrashLogs.zip" >nul 2>&1
)

if exist "C:\Temp\CrashLogs.zip" (
    powershell.exe -NoProfile -Command "Write-Host '[FAIL] Could not remove the previous C:\Temp\CrashLogs.zip. Close or delete it before running this script.' -ForegroundColor Red" 2>nul
    exit /b 1
)

mkdir "%OUTPUT%" >nul 2>&1
mkdir "%OUTPUT%\Dumps" >nul 2>&1

if not exist "%OUTPUT%" (
    powershell.exe -NoProfile -Command "Write-Host '[FAIL] Could not create the CrashLogs output folder.' -ForegroundColor Red" 2>nul
    exit /b 1
)

type nul >"%ERRORLOG%"
type nul >"%ROBOLOG%"

> "%SUMMARY%" (
    echo ============================================================
    echo COLLECT DEVICE CRASH LOGS - VERBOSE RUN LOG
    echo ============================================================
    echo Collection started: %DATE% %TIME%
    echo Computer name: %COMPUTERNAME%
    echo Running account: %USERDOMAIN%\%USERNAME%
    echo Output folder: %OUTPUT%
    echo.
)

call :Log INFO "Administrator rights confirmed."
call :Log INFO "Previous output cleaned successfully."
call :Log INFO "Beginning pre-flight checks."

rem ------------------------------------------------------------
rem Pre-flight disk space / dump size check
rem ------------------------------------------------------------

set "PREFLIGHTTMP=%TEMP%\CrashLogs-Preflight-%RANDOM%.tmp"

rem Keep embedded PowerShell on one physical CMD line. Multi-line -Command blocks
rem using caret continuation can be parsed inconsistently by cmd.exe.
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "try { $ErrorActionPreference='Stop'; $free=[int64](Get-PSDrive -Name C).Free; $paths=@('C:\Windows\MEMORY.DMP','C:\Windows\Minidump','C:\Windows\LiveKernelReports'); $sum=[int64]0; foreach($p in $paths){ if(Test-Path -LiteralPath $p){ $item=Get-Item -LiteralPath $p -Force; if($item.PSIsContainer){ $m=Get-ChildItem -LiteralPath $p -File -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum; if($null -ne $m.Sum){$sum += [int64]$m.Sum} } else {$sum += [int64]$item.Length} } }; $required=$sum+(2GB); Write-Output ('FREE_GB=' + ('{0:N2}' -f ($free/1GB))); Write-Output ('DUMP_GB=' + ('{0:N2}' -f ($sum/1GB))); Write-Output ('REQUIRED_GB=' + ('{0:N2}' -f ($required/1GB))); Write-Output ('LOW_SPACE=' + [int]($free -lt $required)); exit 0 } catch { Write-Error $_; exit 1 }" > "%PREFLIGHTTMP%" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if not !RC! EQU 0 (
    call :Log WARN "Disk-space pre-flight check failed with exit code !RC!. Collection will continue. See Collection-Errors.txt."
)

if exist "%PREFLIGHTTMP%" (
    for /f "usebackq tokens=1,* delims==" %%A in ("!PREFLIGHTTMP!") do (
        set "%%A=%%B"
    )
    del /f /q "%PREFLIGHTTMP%" >nul 2>&1
)

if defined FREE_GB (
    call :Log INFO "Free space on C: !FREE_GB! GB."
) else (
    call :Log WARN "Could not determine free disk space."
)

if defined DUMP_GB (
    call :Log INFO "Existing crash dump data found: !DUMP_GB! GB."
)

if "!LOW_SPACE!"=="1" (
    call :Log WARN "Free space is below the conservative recommendation of !REQUIRED_GB! GB for copying the crash data. Collection will continue, but large dump copies may fail."
) else if defined REQUIRED_GB (
    call :Log OK "Disk space pre-flight passed. Conservative requirement: !REQUIRED_GB! GB."
)

rem ------------------------------------------------------------
rem Windows Event Logs
rem ------------------------------------------------------------

call :Log INFO "Exporting Windows event logs..."

call :ExportEventLog "System" "System.evtx" "System event log" REQUIRED
call :ExportEventLog "Application" "Application.evtx" "Application event log" REQUIRED
call :ExportEventLog "Microsoft-Windows-Diagnostics-Performance/Operational" "Diagnostics-Performance.evtx" "Diagnostics-Performance event log" OPTIONAL
call :ExportEventLog "Microsoft-Windows-Power-Troubleshooter/Operational" "Power-Troubleshooter.evtx" "Power-Troubleshooter event log" OPTIONAL

rem ------------------------------------------------------------
rem Power information
rem ------------------------------------------------------------

call :Log INFO "Collecting power and sleep information..."

call :RunPowerCfg "/a" "powercfg-a.txt" "Available sleep states"
call :RunPowerCfg "/lastwake" "lastwake.txt" "Last wake information"
call :RunPowerCfg "/waketimers" "waketimers.txt" "Wake timers"
call :RunPowerCfg "/requests" "power-requests.txt" "Active power requests"
call :RunPowerCfg "/devicequery wake_armed" "wake-armed-devices.txt" "Wake-armed devices"

call :Log INFO "Attempting SleepStudy collection..."
powercfg /sleepstudy /output "%OUTPUT%\sleepstudy.html" >nul 2>>"%ERRORLOG%"
set "RC=!errorlevel!"

if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\sleepstudy.html" "SleepStudy report"
) else (
    call :Log SKIP "SleepStudy is unavailable or unsupported on this device. Other power diagnostics were still collected."
)

rem ------------------------------------------------------------
rem System information
rem ------------------------------------------------------------

call :Log INFO "Collecting system information..."

systeminfo >"%OUTPUT%\systeminfo.txt" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\systeminfo.txt" "System information"
) else (
    call :Log FAIL "systeminfo failed with exit code !RC!."
)

driverquery /v >"%OUTPUT%\drivers.txt" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\drivers.txt" "Driver inventory"
) else (
    call :Log FAIL "driverquery failed with exit code !RC!."
)

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_QuickFixEngineering -ErrorAction Stop | Select-Object HotFixID,Description,InstalledBy,InstalledOn | Format-Table -AutoSize | Out-File -LiteralPath ($env:OUTPUT + '\windows-updates.txt') -Encoding utf8 -Width 4096" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\windows-updates.txt" "Windows update inventory"
) else (
    call :Log FAIL "Windows update inventory collection failed with exit code !RC!."
)

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_BIOS -ErrorAction Stop | Select-Object SerialNumber,SMBIOSBIOSVersion,ReleaseDate | Format-List | Out-File -LiteralPath ($env:OUTPUT + '\bios.txt') -Encoding utf8 -Width 4096" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\bios.txt" "BIOS information"
) else (
    call :Log FAIL "BIOS information collection failed with exit code !RC!."
)

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_ComputerSystem -ErrorAction Stop | Select-Object Manufacturer,Model,Name,UserName,TotalPhysicalMemory | Format-List | Out-File -LiteralPath ($env:OUTPUT + '\model.txt') -Encoding utf8 -Width 4096" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\model.txt" "Computer model information"
) else (
    call :Log FAIL "Computer model information collection failed with exit code !RC!."
)

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_OperatingSystem -ErrorAction Stop | Select-Object Caption,Version,BuildNumber,LastBootUpTime | Format-List | Out-File -LiteralPath ($env:OUTPUT + '\os.txt') -Encoding utf8 -Width 4096" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"
if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\os.txt" "Operating system information"
) else (
    call :Log FAIL "Operating system information collection failed with exit code !RC!."
)

rem ------------------------------------------------------------
rem Crash dump configuration
rem ------------------------------------------------------------

call :Log INFO "Collecting crash dump configuration and page file information..."

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "try { $dest=Join-Path $env:OUTPUT 'Crash-Dump-Configuration.txt'; $ccPath='HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl'; $cc=Get-ItemProperty -LiteralPath $ccPath -ErrorAction Stop; $dumpType=switch([int]$cc.CrashDumpEnabled){0{'None'};1{if($cc.FilterPages -eq 1){'Active memory dump'}else{'Complete memory dump'}};2{'Kernel memory dump'};3{'Small memory dump'};7{'Automatic memory dump'};default{'Unknown / value ' + $cc.CrashDumpEnabled}}; @('=== Crash dump summary ===',('Crash dump type: ' + $dumpType),('CrashDumpEnabled: ' + $cc.CrashDumpEnabled),('DumpFile: ' + $cc.DumpFile),('MinidumpDir: ' + $cc.MinidumpDir),('AutoReboot: ' + $cc.AutoReboot),('Overwrite: ' + $cc.Overwrite),('LogEvent: ' + $cc.LogEvent),('FilterPages: ' + $cc.FilterPages),'') | Out-File -LiteralPath $dest -Encoding utf8 -Width 4096; '=== Win32_OSRecoveryConfiguration ===' | Out-File -LiteralPath $dest -Append -Encoding utf8; Get-CimInstance Win32_OSRecoveryConfiguration -ErrorAction Stop | Select-Object AutoReboot,DebugInfoType,DebugFilePath,MiniDumpDirectory,OverwriteExistingDebugFile,WriteToSystemLog | Format-List | Out-File -LiteralPath $dest -Append -Encoding utf8 -Width 4096; '=== Raw CrashControl registry values ===' | Out-File -LiteralPath $dest -Append -Encoding utf8; $cc | Select-Object CrashDumpEnabled,DumpFile,MinidumpDir,AutoReboot,Overwrite,LogEvent,FilterPages,AlwaysKeepMemoryDump,DedicatedDumpFile,DumpFileSize | Format-List | Out-File -LiteralPath $dest -Append -Encoding utf8 -Width 4096; '=== Page file configuration ===' | Out-File -LiteralPath $dest -Append -Encoding utf8; Get-CimInstance Win32_ComputerSystem -ErrorAction SilentlyContinue | Select-Object AutomaticManagedPagefile | Format-List | Out-File -LiteralPath $dest -Append -Encoding utf8 -Width 4096; Get-CimInstance Win32_PageFileSetting -ErrorAction SilentlyContinue | Select-Object Name,InitialSize,MaximumSize | Format-Table -AutoSize | Out-File -LiteralPath $dest -Append -Encoding utf8 -Width 4096; Get-CimInstance Win32_PageFileUsage -ErrorAction SilentlyContinue | Select-Object Name,AllocatedBaseSize,CurrentUsage,PeakUsage | Format-Table -AutoSize | Out-File -LiteralPath $dest -Append -Encoding utf8 -Width 4096; exit 0 } catch { Write-Error $_; exit 1 }" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"

if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\Crash-Dump-Configuration.txt" "Crash dump configuration and page file information"
) else (
    call :Log FAIL "Crash dump configuration collection failed with exit code !RC!."
)

rem ------------------------------------------------------------
rem Reliability history
rem ------------------------------------------------------------

call :Log INFO "Collecting up to 500 reliability history records from the last 90 days..."

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "try { $dest=Join-Path $env:OUTPUT 'Reliability-History.txt'; $cutoff=(Get-Date).AddDays(-90); $records=Get-CimInstance Win32_ReliabilityRecords -ErrorAction Stop | Where-Object {$_.TimeGenerated -ge $cutoff} | Sort-Object TimeGenerated -Descending | Select-Object -First 500 TimeGenerated,SourceName,EventIdentifier,ProductName,Message; if($records){$records | Format-List | Out-File -LiteralPath $dest -Encoding utf8 -Width 4096}else{'No reliability history records were returned for the last 90 days.' | Out-File -LiteralPath $dest -Encoding utf8}; exit 0 } catch { Write-Error $_; exit 2 }" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"

if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\Reliability-History.txt" "Reliability history"
) else (
    call :Log SKIP "Reliability history could not be collected. The provider may be unavailable or disabled. See Collection-Errors.txt for details."
)

rem ------------------------------------------------------------
rem Crash dump files
rem ------------------------------------------------------------

call :Log INFO "Collecting crash dump files..."

call :CollectDumpFolder "C:\Windows\Minidump" "%OUTPUT%\Dumps\Minidump" "Windows minidumps"
call :CollectDumpFolder "C:\Windows\LiveKernelReports" "%OUTPUT%\Dumps\LiveKernelReports" "LiveKernelReports"

if exist "C:\Windows\MEMORY.DMP" (
    for /f "usebackq delims=" %%A in (`powershell.exe -NoProfile -Command "'{0:N2} GB' -f ((Get-Item -LiteralPath 'C:\Windows\MEMORY.DMP' -Force).Length/1GB)"`) do set "MEMDUMPSIZE=%%A"
    call :Log INFO "MEMORY.DMP found. Size: !MEMDUMPSIZE!."
    call :Log INFO "Copying MEMORY.DMP with Robocopy..."

    robocopy "C:\Windows" "%OUTPUT%\Dumps" "MEMORY.DMP" /ZB /J /R:2 /W:2 /COPY:DAT /NP /NFL /NDL /NJH /NJS /LOG+:"%ROBOLOG%" >nul 2>&1
    set "RC=!errorlevel!"

    if !RC! GEQ 8 (
        call :Log FAIL "MEMORY.DMP copy failed. Robocopy exit code !RC!. See Robocopy.log."
    ) else (
        if exist "%OUTPUT%\Dumps\MEMORY.DMP" (
            set "DUMPFOUND=1"
            call :Log OK "MEMORY.DMP copied successfully."
        ) else (
            call :Log FAIL "Robocopy did not report a failure, but MEMORY.DMP is missing from the output folder."
        )
    )
) else (
    call :Log SKIP "No C:\Windows\MEMORY.DMP file is present."
)

if "!DUMPFOUND!"=="0" (
    >"%OUTPUT%\Dumps\No-Dumps-Found.txt" (
        echo No crash dump files were present when this collection ran.
        echo.
        echo Checked:
        echo   C:\Windows\Minidump
        echo   C:\Windows\LiveKernelReports
        echo   C:\Windows\MEMORY.DMP
        echo.
        echo This is not automatically an error. The device may simply not have
        echo generated a Windows crash dump. Review Crash-Dump-Configuration.txt
        echo and the exported event logs for additional context.
        echo.
        echo Collection time: !DATE! !TIME!
    )
    call :Log INFO "No crash dump files were found. Added Dumps\No-Dumps-Found.txt to make this explicit."
)

rem ------------------------------------------------------------
rem Measure collected data
rem ------------------------------------------------------------

set "COLLECTEDDISPLAY=Unknown"

for /f "usebackq delims=" %%A in (`powershell.exe -NoProfile -Command "$m=Get-ChildItem -LiteralPath $env:OUTPUT -File -Recurse -Force -ErrorAction SilentlyContinue | Measure-Object -Property Length -Sum; if($null -eq $m.Sum){$b=0}else{$b=$m.Sum}; if($b -ge 1GB){'{0:N2} GB' -f ($b/1GB)}elseif($b -ge 1MB){'{0:N2} MB' -f ($b/1MB)}else{'{0:N2} KB' -f ($b/1KB)}"`) do set "COLLECTEDDISPLAY=%%A"

call :Log INFO "Total collected data: !COLLECTEDDISPLAY!."

call :WriteFinalSummary

echo.
if !FAILCOUNT! GTR 0 (
    call :Console WARN "Collection completed, but one or more collection items failed. Review Collection-Summary.txt and Collection-Errors.txt."
) else if !WARNCOUNT! GTR 0 (
    call :Console WARN "Collection completed successfully with warnings. Review Collection-Summary.txt for details."
) else (
    call :Console OK "Collection completed successfully with no reported failures."
)

call :Console INFO "Collected files are here: %OUTPUT%"
call :Console OK "Manually compress the CrashLogs folder, then upload the resulting ZIP to the ticket."

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Crash log collection is complete. The files are in C:\Temp\CrashLogs. Manually compress the CrashLogs folder to a ZIP, then upload that ZIP to the ticket. Review Collection-Summary.txt if any warnings were shown.','Crash logs collected','OK','Information')" 2>nul

exit /b 0

rem ============================================================
rem Subroutines
rem ============================================================

:ExportEventLog
set "CHANNEL=%~1"
set "FILENAME=%~2"
set "LABEL=%~3"
set "IMPORTANCE=%~4"

call :Log INFO "Checking %LABEL%..."

wevtutil gl "%CHANNEL%" >nul 2>&1
set "RC=!errorlevel!"

if not !RC! EQU 0 (
    if /I "%IMPORTANCE%"=="REQUIRED" (
        call :Log FAIL "%LABEL% is not available on this device."
    ) else (
        call :Log SKIP "%LABEL% is not available on this device."
    )
    exit /b
)

wevtutil epl "%CHANNEL%" "%OUTPUT%\%FILENAME%" /ow:true 2>>"%ERRORLOG%"
set "RC=!errorlevel!"

if not !RC! EQU 0 (
    call :Log FAIL "%LABEL% export failed with exit code !RC!."
    exit /b
)

call :ValidateOutput "%OUTPUT%\%FILENAME%" "%LABEL%"

exit /b


:RunPowerCfg
set "PCARGS=%~1"
set "PCFILE=%~2"
set "PCLABEL=%~3"

call :Log INFO "Collecting %PCLABEL%..."

powercfg %PCARGS% >"%OUTPUT%\%PCFILE%" 2>>"%ERRORLOG%"
set "RC=!errorlevel!"

if !RC! EQU 0 (
    call :ValidateOutput "%OUTPUT%\%PCFILE%" "%PCLABEL%"
) else (
    call :Log FAIL "%PCLABEL% collection failed with exit code !RC!."
)

exit /b


:ValidateOutput
set "CHECKFILE=%~1"
set "CHECKLABEL=%~2"
set "CHECKSIZE="

if not exist "!CHECKFILE!" (
    call :Log FAIL "!CHECKLABEL! command completed but no output file was created."
    exit /b
)

for %%F in ("!CHECKFILE!") do set "CHECKSIZE=%%~zF"

if not defined CHECKSIZE (
    call :Log FAIL "!CHECKLABEL! output exists but its size could not be validated."
) else if !CHECKSIZE! LEQ 0 (
    call :Log FAIL "!CHECKLABEL! created an empty output file."
) else (
    call :Log OK "!CHECKLABEL! collected successfully. Output size: !CHECKSIZE! bytes."
)

exit /b


:CollectDumpFolder
set "SRC=%~1"
set "DST=%~2"
set "LABEL=%~3"

if not exist "%SRC%" (
    call :Log SKIP "%LABEL% folder is not present."
    exit /b
)

set "SRC_COUNT=0"
for /f %%C in ('dir /a-d /s /b "%SRC%\*" 2^>nul ^| find /c /v ""') do set "SRC_COUNT=%%C"

if "!SRC_COUNT!"=="0" (
    call :Log SKIP "%LABEL% folder exists but contains no files."
    exit /b
)

call :Log INFO "Copying %LABEL% - !SRC_COUNT! source file(s) found..."

robocopy "%SRC%" "%DST%" /E /ZB /R:2 /W:2 /COPY:DAT /DCOPY:DAT /XJ /NP /NFL /NDL /NJH /NJS /LOG+:"%ROBOLOG%" >nul 2>&1
set "RC=!errorlevel!"

if !RC! GEQ 8 (
    call :Log FAIL "%LABEL% copy reported a Robocopy failure. Exit code !RC!. See Robocopy.log."
    exit /b
)

set "DST_COUNT=0"
if exist "%DST%" (
    for /f %%C in ('dir /a-d /s /b "%DST%\*" 2^>nul ^| find /c /v ""') do set "DST_COUNT=%%C"
)

if !DST_COUNT! EQU 0 (
    call :Log FAIL "%LABEL% source contained !SRC_COUNT! file(s), but no files were found in the destination after Robocopy."
) else if not !DST_COUNT! EQU !SRC_COUNT! (
    call :Log WARN "%LABEL% Robocopy completed without a failure code, but the file counts changed during collection. Source count before copy: !SRC_COUNT!. Destination count after copy: !DST_COUNT!. See Robocopy.log if investigation requires every file."
) else (
    set "DUMPFOUND=1"
    call :Log OK "%LABEL% copied successfully. Files copied/present: !DST_COUNT!."
)

exit /b


:Log
set "LOGLEVEL=%~1"
set "LOGMESSAGE=%~2"
set "LOGSTAMP=!DATE! !TIME:~0,8!"

>>"%SUMMARY%" echo [!LOGSTAMP!] [!LOGLEVEL!] !LOGMESSAGE!

if /I "!LOGLEVEL!"=="OK" set /a OKCOUNT+=1 >nul
if /I "!LOGLEVEL!"=="WARN" set /a WARNCOUNT+=1 >nul
if /I "!LOGLEVEL!"=="SKIP" set /a SKIPCOUNT+=1 >nul
if /I "!LOGLEVEL!"=="FAIL" set /a FAILCOUNT+=1 >nul

call :Console "!LOGLEVEL!" "!LOGMESSAGE!"
exit /b


:Console
set "CONSOLELEVEL=%~1"
set "CONSOLEMESSAGE=%~2"
set "COLLECT_LOG_LEVEL=!CONSOLELEVEL!"
set "COLLECT_LOG_MESSAGE=!CONSOLEMESSAGE!"

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "$level=$env:COLLECT_LOG_LEVEL; $msg=$env:COLLECT_LOG_MESSAGE; $colors=@{INFO='Cyan';OK='Green';WARN='Yellow';SKIP='DarkYellow';FAIL='Red'}; $color=$colors[$level]; if($null -eq $color){$color='Gray'}; Write-Host ('['+$level+'] '+$msg) -ForegroundColor $color" 2>nul

if errorlevel 1 echo [!CONSOLELEVEL!] !CONSOLEMESSAGE!
exit /b


:WriteFinalSummary
set "FINISHEDSTAMP=!DATE! !TIME!"

>>"%SUMMARY%" (
    echo.
    echo ============================================================
    echo FINAL SUMMARY
    echo ============================================================
    echo Collection finished: !FINISHEDSTAMP!
    echo Successful operations: !OKCOUNT!
    echo Warnings: !WARNCOUNT!
    echo Skipped / not applicable: !SKIPCOUNT!
    echo Failed operations: !FAILCOUNT!
    echo Collected data size: !COLLECTEDDISPLAY!
    echo Output folder: %OUTPUT%
    echo Compression performed by script: No
    echo Technician action: Manually compress the CrashLogs folder before upload.
    echo.
    echo Detailed command errors: %ERRORLOG%
    echo Robocopy details: %ROBOLOG%
    echo ============================================================
)

exit /b

```
