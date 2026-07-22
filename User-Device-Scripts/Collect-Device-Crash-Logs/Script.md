# Collect Device Crash Logs

Use this script when investigating blue screens, random restarts, unexpected shutdowns, freezes, driver crashes, or sleep and wake problems.

For the investigation workflow, see [README.md](./README.md).

> **Important:** Run this as administrator. Crash dumps can contain sensitive data and can make the output ZIP very large.

## Output

```text
C:\Temp\CrashLogs.zip
```

## Steps

1. Copy the complete script below into Notepad.
2. Save it as `Collect-Crash-Logs.cmd`.
3. Right-click the saved file and select **Run as administrator**.
4. Wait for the confirmation popup.
5. Upload `C:\Temp\CrashLogs.zip` to the approved ticket or support location.

Do not paste this script line-by-line into an open Command Prompt.

The script automatically closes its elevated Command Prompt on success or failure.

## Script

```bat
@echo off
setlocal EnableExtensions

set "OUTPUT=C:\Temp\CrashLogs"
set "ZIPFILE=C:\Temp\CrashLogs.zip"
set "COMPRESSIONLOG=C:\Temp\CrashLogs-Compression-Error.txt"

echo Checking administrator rights...
fltmc >nul 2>&1

if errorlevel 1 (
    echo.
    echo ERROR: This script must be run as administrator.
    start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('This script must be run as administrator. Right-click it and select Run as administrator.','Administrator rights required','OK','Error')"
    exit
)

echo Creating output folder...
mkdir "C:\Temp" 2>nul
rmdir /s /q "%OUTPUT%" 2>nul
del /f /q "%ZIPFILE%" 2>nul
del /f /q "%COMPRESSIONLOG%" 2>nul
mkdir "%OUTPUT%" 2>nul

echo Collection started: %DATE% %TIME% >"%OUTPUT%\Collection-Summary.txt"
echo Computer name: %COMPUTERNAME% >>"%OUTPUT%\Collection-Summary.txt"
echo Running account: %USERDOMAIN%\%USERNAME% >>"%OUTPUT%\Collection-Summary.txt"

echo Exporting Windows event logs...
wevtutil epl System "%OUTPUT%\System.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl Application "%OUTPUT%\Application.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Diagnostics-Performance/Operational" "%OUTPUT%\Diagnostics-Performance.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Power-Troubleshooter/Operational" "%OUTPUT%\Power-Troubleshooter.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting power information...
powercfg /a >"%OUTPUT%\powercfg-a.txt" 2>&1
powercfg /lastwake >"%OUTPUT%\lastwake.txt" 2>&1
powercfg /waketimers >"%OUTPUT%\waketimers.txt" 2>&1
powercfg /requests >"%OUTPUT%\power-requests.txt" 2>&1
powercfg /devicequery wake_armed >"%OUTPUT%\wake-armed-devices.txt" 2>&1
powercfg /sleepstudy /output "%OUTPUT%\sleepstudy.html" 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting system information...
systeminfo >"%OUTPUT%\systeminfo.txt" 2>&1
driverquery /v >"%OUTPUT%\drivers.txt" 2>&1
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_QuickFixEngineering | Select-Object HotFixID,Description,InstalledBy,InstalledOn | Format-Table -AutoSize | Out-File -FilePath '%OUTPUT%\windows-updates.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_BIOS | Select-Object SerialNumber,SMBIOSBIOSVersion,ReleaseDate | Format-List | Out-File -FilePath '%OUTPUT%\bios.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer,Model,Name,UserName,TotalPhysicalMemory | Format-List | Out-File -FilePath '%OUTPUT%\model.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_OperatingSystem | Select-Object Caption,Version,BuildNumber,LastBootUpTime | Format-List | Out-File -FilePath '%OUTPUT%\os.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting crash dump files...
mkdir "%OUTPUT%\Dumps" 2>nul

if exist "C:\Windows\Minidump" (
    xcopy "C:\Windows\Minidump\*" "%OUTPUT%\Dumps\Minidump\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

if exist "C:\Windows\LiveKernelReports" (
    xcopy "C:\Windows\LiveKernelReports\*" "%OUTPUT%\Dumps\LiveKernelReports\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

if exist "C:\Windows\MEMORY.DMP" (
    copy /Y "C:\Windows\MEMORY.DMP" "%OUTPUT%\Dumps\" >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

echo Creating ZIP file...
where tar.exe >nul 2>&1

if not errorlevel 1 (
    tar.exe -a -c -f "%ZIPFILE%" -C "%OUTPUT%" . 2>"%COMPRESSIONLOG%"
) else (
    powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "try { Compress-Archive -Path '%OUTPUT%\*' -DestinationPath '%ZIPFILE%' -Force -ErrorAction Stop; exit 0 } catch { Write-Error $_; exit 1 }" 2>"%COMPRESSIONLOG%"
)

if errorlevel 1 goto ZIPFAILED
if not exist "%ZIPFILE%" goto ZIPFAILED

del /f /q "%COMPRESSIONLOG%" 2>nul

echo.
echo Done. Upload this file to the ticket:
echo %ZIPFILE%

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Crash log collection is complete. Upload C:\Temp\CrashLogs.zip to the ticket.','Crash logs collected','OK','Information')"

exit

rem Success exit intentionally has another physical line after it.

:ZIPFAILED
echo.
echo ERROR: The logs were collected, but ZIP creation failed.
echo The uncompressed files are available here:
echo %OUTPUT%

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('ZIP creation failed. The uncompressed crash logs are available at C:\Temp\CrashLogs.','Crash log collection error','OK','Error')"

exit

rem End of script
```
