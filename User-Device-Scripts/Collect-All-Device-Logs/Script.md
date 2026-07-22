# Collect All Device Diagnostic Logs

Use this combined collector when:

- The ticket involves both crashes and Intune app behaviour
- The original symptom is unclear
- A failed deployment may be related to restarts, disk, drivers, security controls, or power events
- Escalation requires one complete diagnostic package

Use the dedicated collectors when only one area is required:

- [Crash-only collector](../Collect-Device-Crash-Logs/Script.md)
- [Intune-only collector](../Collect-Intune-App-Logs/Script.md)

Troubleshooting guides:

- [Crash troubleshooting](../Collect-Device-Crash-Logs/README.md)
- [Intune app troubleshooting](../Collect-Intune-App-Logs/README.md)

> **Important:** Run this as administrator. This is the largest collection option and may include a multi-gigabyte memory dump.

## Output

```text
C:\Temp\DeviceDiagnosticLogs.zip
```

## Steps

1. Copy the complete script below into Notepad.
2. Save it as `Collect-All-Device-Logs.cmd`.
3. Right-click the saved file and select **Run as administrator**.
4. Wait for the confirmation popup.
5. Upload `C:\Temp\DeviceDiagnosticLogs.zip` to the approved ticket or support location.

Do not paste this script line-by-line into an open Command Prompt.

The script automatically closes its elevated Command Prompt on success or failure.

## Script

```bat
@echo off
setlocal EnableExtensions

set "OUTPUT=C:\Temp\DeviceDiagnosticLogs"
set "ZIPFILE=C:\Temp\DeviceDiagnosticLogs.zip"
set "COMPRESSIONLOG=C:\Temp\DeviceDiagnosticLogs-Compression-Error.txt"

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
mkdir "%OUTPUT%\Crash" 2>nul
mkdir "%OUTPUT%\Crash\Dumps" 2>nul
mkdir "%OUTPUT%\Intune" 2>nul
mkdir "%OUTPUT%\Intune\Event-Logs" 2>nul
mkdir "%OUTPUT%\Intune\CompanyPortal" 2>nul
mkdir "%OUTPUT%\System-Info" 2>nul

echo Collection started: %DATE% %TIME% >"%OUTPUT%\Collection-Summary.txt"
echo Computer name: %COMPUTERNAME% >>"%OUTPUT%\Collection-Summary.txt"
echo Running account: %USERDOMAIN%\%USERNAME% >>"%OUTPUT%\Collection-Summary.txt"

echo Exporting Windows crash event logs...
wevtutil epl System "%OUTPUT%\Crash\System.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl Application "%OUTPUT%\Crash\Application.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Diagnostics-Performance/Operational" "%OUTPUT%\Crash\Diagnostics-Performance.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Power-Troubleshooter/Operational" "%OUTPUT%\Crash\Power-Troubleshooter.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting power information...
powercfg /a >"%OUTPUT%\Crash\powercfg-a.txt" 2>&1
powercfg /lastwake >"%OUTPUT%\Crash\lastwake.txt" 2>&1
powercfg /waketimers >"%OUTPUT%\Crash\waketimers.txt" 2>&1
powercfg /requests >"%OUTPUT%\Crash\power-requests.txt" 2>&1
powercfg /devicequery wake_armed >"%OUTPUT%\Crash\wake-armed-devices.txt" 2>&1
powercfg /sleepstudy /output "%OUTPUT%\Crash\sleepstudy.html" 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting crash dump files...
if exist "C:\Windows\Minidump" (
    xcopy "C:\Windows\Minidump\*" "%OUTPUT%\Crash\Dumps\Minidump\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

if exist "C:\Windows\LiveKernelReports" (
    xcopy "C:\Windows\LiveKernelReports\*" "%OUTPUT%\Crash\Dumps\LiveKernelReports\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

if exist "C:\Windows\MEMORY.DMP" (
    copy /Y "C:\Windows\MEMORY.DMP" "%OUTPUT%\Crash\Dumps\" >nul 2>>"%OUTPUT%\Collection-Errors.txt"
)

echo Collecting Intune Management Extension logs...
if exist "%ProgramData%\Microsoft\IntuneManagementExtension\Logs" (
    xcopy "%ProgramData%\Microsoft\IntuneManagementExtension\Logs\*" "%OUTPUT%\Intune\IME-Logs\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
) else (
    echo Intune Management Extension log folder was not found. >"%OUTPUT%\Intune\IME-Logs-Not-Found.txt"
)

echo Collecting Company Portal logs from local user profiles...
for /d %%U in ("C:\Users\*") do (
    if exist "%%U\AppData\Local\Packages\Microsoft.CompanyPortal_8wekyb3d8bbwe\LocalState" (
        mkdir "%OUTPUT%\Intune\CompanyPortal\%%~nxU" 2>nul
        xcopy "%%U\AppData\Local\Packages\Microsoft.CompanyPortal_8wekyb3d8bbwe\LocalState\Log_*.log" "%OUTPUT%\Intune\CompanyPortal\%%~nxU\" /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
    )
)

echo Exporting Intune and app deployment event logs...
wevtutil epl "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" "%OUTPUT%\Intune\Event-Logs\Intune-MDM-Admin.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppXDeploymentServer/Operational" "%OUTPUT%\Intune\Event-Logs\AppX-Deployment.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppxPackaging/Operational" "%OUTPUT%\Intune\Event-Logs\AppX-Packaging.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Bits-Client/Operational" "%OUTPUT%\Intune\Event-Logs\BITS-Client.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-DeliveryOptimization/Operational" "%OUTPUT%\Intune\Event-Logs\Delivery-Optimization.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-CodeIntegrity/Operational" "%OUTPUT%\Intune\Event-Logs\Code-Integrity.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Windows Defender/Operational" "%OUTPUT%\Intune\Event-Logs\Defender-Operational.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppLocker/EXE and DLL" "%OUTPUT%\Intune\Event-Logs\AppLocker-EXE-DLL.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppLocker/MSI and Script" "%OUTPUT%\Intune\Event-Logs\AppLocker-MSI-Script.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting system and Intune state...
systeminfo >"%OUTPUT%\System-Info\systeminfo.txt" 2>&1
driverquery /v >"%OUTPUT%\System-Info\drivers.txt" 2>&1
sc query IntuneManagementExtension >"%OUTPUT%\Intune\IME-Service.txt" 2>&1
sc qc IntuneManagementExtension >>"%OUTPUT%\Intune\IME-Service.txt" 2>&1
dsregcmd /status >"%OUTPUT%\Intune\dsregcmd-status.txt" 2>&1
ipconfig /all >"%OUTPUT%\Intune\ipconfig-all.txt" 2>&1
netsh winhttp show proxy >"%OUTPUT%\Intune\winhttp-proxy.txt" 2>&1
whoami /all >"%OUTPUT%\System-Info\running-account.txt" 2>&1

powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_QuickFixEngineering | Select-Object HotFixID,Description,InstalledBy,InstalledOn | Format-Table -AutoSize | Out-File -FilePath '%OUTPUT%\System-Info\windows-updates.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_BIOS | Select-Object SerialNumber,SMBIOSBIOSVersion,ReleaseDate | Format-List | Out-File -FilePath '%OUTPUT%\System-Info\bios.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_ComputerSystem | Select-Object Manufacturer,Model,Name,UserName,TotalPhysicalMemory | Format-List | Out-File -FilePath '%OUTPUT%\System-Info\model.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_OperatingSystem | Select-Object Caption,Version,BuildNumber,LastBootUpTime | Format-List | Out-File -FilePath '%OUTPUT%\System-Info\os.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_LogicalDisk | Select-Object DeviceID,VolumeName,FileSystem,Size,FreeSpace | Format-Table -AutoSize | Out-File -FilePath '%OUTPUT%\System-Info\disk-space.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | Where-Object DisplayName | Select-Object DisplayName,DisplayVersion,Publisher,InstallDate,InstallLocation,UninstallString,QuietUninstallString | Sort-Object DisplayName | Export-Csv -Path '%OUTPUT%\Intune\Installed-Apps-Machine.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-AppxPackage -AllUsers | Select-Object Name,PackageFullName,Version,Architecture,Publisher,InstallLocation,PackageUserInformation | Sort-Object Name | Export-Csv -Path '%OUTPUT%\Intune\Installed-Appx-All-Users.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-ScheduledTask | Where-Object { $_.TaskPath -like '\Microsoft\Windows\EnterpriseMgmt\*' -or $_.TaskName -match 'Intune' } | Select-Object TaskPath,TaskName,State,Author,Description | Sort-Object TaskPath,TaskName | Export-Csv -Path '%OUTPUT%\Intune\Intune-Scheduled-Tasks.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"

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

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Device diagnostic log collection is complete. Upload C:\Temp\DeviceDiagnosticLogs.zip to the ticket.','Device logs collected','OK','Information')"

exit

rem Success exit intentionally has another physical line after it.

:ZIPFAILED
echo.
echo ERROR: The logs were collected, but ZIP creation failed.
echo The uncompressed files are available here:
echo %OUTPUT%

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('ZIP creation failed. The uncompressed logs are available at C:\Temp\DeviceDiagnosticLogs.','Device log collection error','OK','Error')"

exit

rem End of script
```
