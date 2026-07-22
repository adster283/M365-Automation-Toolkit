# Collect Intune App Deployment Logs

Use this script when an Intune app does not install, reports the wrong state, is stuck, or was unexpectedly removed.

For the complete troubleshooting workflow, see [README.md](./README.md).

> **Important:** Run this as administrator. The output may contain usernames, tenant information, app IDs, commands, and application inventory.

## Output

```text
C:\Temp\IntuneAppLogs.zip
```

## Steps

1. Copy the complete script below into Notepad.
2. Save it as `Collect-Intune-App-Logs.cmd`.
3. Right-click the saved file and select **Run as administrator**.
4. Wait for the confirmation popup.
5. Upload `C:\Temp\IntuneAppLogs.zip` to the approved ticket or support location.

Do not paste this script line-by-line into an open Command Prompt.

The script automatically closes its elevated Command Prompt on success or failure.

## Script

```bat
@echo off
setlocal EnableExtensions

set "OUTPUT=C:\Temp\IntuneAppLogs"
set "ZIPFILE=C:\Temp\IntuneAppLogs.zip"
set "COMPRESSIONLOG=C:\Temp\IntuneAppLogs-Compression-Error.txt"

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
mkdir "%OUTPUT%\Event-Logs" 2>nul
mkdir "%OUTPUT%\CompanyPortal" 2>nul

echo Collection started: %DATE% %TIME% >"%OUTPUT%\Collection-Summary.txt"
echo Computer name: %COMPUTERNAME% >>"%OUTPUT%\Collection-Summary.txt"
echo Running account: %USERDOMAIN%\%USERNAME% >>"%OUTPUT%\Collection-Summary.txt"

echo Collecting Intune Management Extension logs...
if exist "%ProgramData%\Microsoft\IntuneManagementExtension\Logs" (
    xcopy "%ProgramData%\Microsoft\IntuneManagementExtension\Logs\*" "%OUTPUT%\IME-Logs\" /E /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
) else (
    echo Intune Management Extension log folder was not found. >"%OUTPUT%\IME-Logs-Not-Found.txt"
)

echo Collecting Company Portal logs from local user profiles...
for /d %%U in ("C:\Users\*") do (
    if exist "%%U\AppData\Local\Packages\Microsoft.CompanyPortal_8wekyb3d8bbwe\LocalState" (
        mkdir "%OUTPUT%\CompanyPortal\%%~nxU" 2>nul
        xcopy "%%U\AppData\Local\Packages\Microsoft.CompanyPortal_8wekyb3d8bbwe\LocalState\Log_*.log" "%OUTPUT%\CompanyPortal\%%~nxU\" /I /H /Y >nul 2>>"%OUTPUT%\Collection-Errors.txt"
    )
)

echo Exporting Intune and application event logs...
wevtutil epl Application "%OUTPUT%\Event-Logs\Application.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-DeviceManagement-Enterprise-Diagnostics-Provider/Admin" "%OUTPUT%\Event-Logs\Intune-MDM-Admin.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppXDeploymentServer/Operational" "%OUTPUT%\Event-Logs\AppX-Deployment.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppxPackaging/Operational" "%OUTPUT%\Event-Logs\AppX-Packaging.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Bits-Client/Operational" "%OUTPUT%\Event-Logs\BITS-Client.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-DeliveryOptimization/Operational" "%OUTPUT%\Event-Logs\Delivery-Optimization.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-CodeIntegrity/Operational" "%OUTPUT%\Event-Logs\Code-Integrity.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-Windows Defender/Operational" "%OUTPUT%\Event-Logs\Defender-Operational.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppLocker/EXE and DLL" "%OUTPUT%\Event-Logs\AppLocker-EXE-DLL.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"
wevtutil epl "Microsoft-Windows-AppLocker/MSI and Script" "%OUTPUT%\Event-Logs\AppLocker-MSI-Script.evtx" /ow:true 2>>"%OUTPUT%\Collection-Errors.txt"

echo Collecting Intune service and device state...
sc query IntuneManagementExtension >"%OUTPUT%\IME-Service.txt" 2>&1
sc qc IntuneManagementExtension >>"%OUTPUT%\IME-Service.txt" 2>&1
dsregcmd /status >"%OUTPUT%\dsregcmd-status.txt" 2>&1
systeminfo >"%OUTPUT%\systeminfo.txt" 2>&1
ipconfig /all >"%OUTPUT%\ipconfig-all.txt" 2>&1
netsh winhttp show proxy >"%OUTPUT%\winhttp-proxy.txt" 2>&1
whoami /all >"%OUTPUT%\running-account.txt" 2>&1

echo Collecting disk and application inventory...
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-CimInstance Win32_LogicalDisk | Select-Object DeviceID,VolumeName,FileSystem,Size,FreeSpace | Format-Table -AutoSize | Out-File -FilePath '%OUTPUT%\disk-space.txt' -Encoding utf8 -Width 4096" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | Where-Object DisplayName | Select-Object DisplayName,DisplayVersion,Publisher,InstallDate,InstallLocation,UninstallString,QuietUninstallString | Sort-Object DisplayName | Export-Csv -Path '%OUTPUT%\Installed-Apps-Machine.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-AppxPackage -AllUsers | Select-Object Name,PackageFullName,Version,Architecture,Publisher,InstallLocation,PackageUserInformation | Sort-Object Name | Export-Csv -Path '%OUTPUT%\Installed-Appx-All-Users.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Get-ScheduledTask | Where-Object { $_.TaskPath -like '\Microsoft\Windows\EnterpriseMgmt\*' -or $_.TaskName -match 'Intune' } | Select-Object TaskPath,TaskName,State,Author,Description | Sort-Object TaskPath,TaskName | Export-Csv -Path '%OUTPUT%\Intune-Scheduled-Tasks.csv' -NoTypeInformation -Encoding UTF8" 2>>"%OUTPUT%\Collection-Errors.txt"

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

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Intune app log collection is complete. Upload C:\Temp\IntuneAppLogs.zip to the ticket.','Intune logs collected','OK','Information')"

exit

rem Success exit intentionally has another physical line after it.

:ZIPFAILED
echo.
echo ERROR: The logs were collected, but ZIP creation failed.
echo The uncompressed files are available here:
echo %OUTPUT%

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('ZIP creation failed. The uncompressed Intune logs are available at C:\Temp\IntuneAppLogs.','Intune log collection error','OK','Error')"

exit

rem End of script
```
