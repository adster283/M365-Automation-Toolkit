# Collect Windows Crash Logs

Use this script to collect Windows crash, sleep, power, driver, update, and system information logs for troubleshooting.

For full details on what this collects and how to review the output, see `README.md`.

> Note: The generated ZIP may contain device names, serial numbers, driver details, Windows update history, event logs, and crash dump files. Handle it as sensitive troubleshooting data.

## Steps

1. Open **Command Prompt as Administrator**.
2. Paste the full script below into Command Prompt.
3. Wait for the confirmation popup.
4. Copy this file from the affected device and attach it to the ticket:

```text
C:\Temp\CrashLogs.zip
```

## Script

```bat
@echo off
echo Creating output folder...
mkdir C:\Temp 2>nul
rmdir /s /q C:\Temp\CrashLogs 2>nul
mkdir C:\Temp\CrashLogs 2>nul

echo Exporting Windows event logs...
wevtutil epl System C:\Temp\CrashLogs\System.evtx
wevtutil epl Application C:\Temp\CrashLogs\Application.evtx

wevtutil epl Microsoft-Windows-Diagnostics-Performance/Operational C:\Temp\CrashLogs\Diagnostics-Performance.evtx 2>nul
wevtutil epl Microsoft-Windows-Power-Troubleshooter/Operational C:\Temp\CrashLogs\Power-Troubleshooter.evtx 2>nul

echo Collecting power information...
powercfg /a > C:\Temp\CrashLogs\powercfg-a.txt
powercfg /lastwake > C:\Temp\CrashLogs\lastwake.txt
powercfg /waketimers > C:\Temp\CrashLogs\waketimers.txt
powercfg /requests > C:\Temp\CrashLogs\power-requests.txt
powercfg /devicequery wake_armed > C:\Temp\CrashLogs\wake-armed-devices.txt
powercfg /sleepstudy /output C:\Temp\CrashLogs\sleepstudy.html 2>nul

echo Collecting system information...
systeminfo > C:\Temp\CrashLogs\systeminfo.txt
driverquery /v > C:\Temp\CrashLogs\drivers.txt
wmic qfe list brief > C:\Temp\CrashLogs\windows-updates.txt
wmic bios get serialnumber,smbiosbiosversion,releasedate > C:\Temp\CrashLogs\bios.txt
wmic computersystem get manufacturer,model,name > C:\Temp\CrashLogs\model.txt
wmic os get caption,version,buildnumber,lastbootuptime > C:\Temp\CrashLogs\os.txt

echo Collecting crash dump files...
mkdir C:\Temp\CrashLogs\Dumps 2>nul
xcopy C:\Windows\Minidump C:\Temp\CrashLogs\Dumps\Minidump /E /I /Y 2>nul
xcopy C:\Windows\LiveKernelReports C:\Temp\CrashLogs\Dumps\LiveKernelReports /E /I /Y 2>nul
copy C:\Windows\MEMORY.DMP C:\Temp\CrashLogs\Dumps\ 2>nul

echo Creating ZIP file...
powershell.exe -NoProfile -ExecutionPolicy Bypass -Command "Compress-Archive -Path 'C:\Temp\CrashLogs\*' -DestinationPath 'C:\Temp\CrashLogs.zip' -Force"

echo.
echo Done. Upload this file to the ticket:
echo C:\Temp\CrashLogs.zip

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Crash log collection complete. Upload this file to the ticket: C:\Temp\CrashLogs.zip','Crash logs collected','OK','Information')"

exit

rem End of script
```
