Open CMD as Administrator

Paste this into the terminal and wait for it to say it has output a file

```
mkdir C:\Temp\CrashLogs 2>nul
 
wevtutil epl System C:\Temp\CrashLogs\System.evtx
wevtutil epl Application C:\Temp\CrashLogs\Application.evtx
 
wevtutil epl Microsoft-Windows-Diagnostics-Performance/Operational C:\Temp\CrashLogs\Diagnostics-Performance.evtx 2>nul
wevtutil epl Microsoft-Windows-Power-Troubleshooter/Operational C:\Temp\CrashLogs\Power-Troubleshooter.evtx 2>nul
 
powercfg /a > C:\Temp\CrashLogs\powercfg-a.txt
powercfg /lastwake > C:\Temp\CrashLogs\lastwake.txt
powercfg /waketimers > C:\Temp\CrashLogs\waketimers.txt
powercfg /requests > C:\Temp\CrashLogs\power-requests.txt
powercfg /devicequery wake_armed > C:\Temp\CrashLogs\wake-armed-devices.txt
powercfg /sleepstudy /output C:\Temp\CrashLogs\sleepstudy.html 2>nul
 
systeminfo > C:\Temp\CrashLogs\systeminfo.txt
driverquery /v > C:\Temp\CrashLogs\drivers.txt
wmic qfe list brief > C:\Temp\CrashLogs\windows-updates.txt
wmic bios get serialnumber,smbiosbiosversion,releasedate > C:\Temp\CrashLogs\bios.txt
wmic computersystem get manufacturer,model,name > C:\Temp\CrashLogs\model.txt
wmic os get caption,version,buildnumber,lastbootuptime > C:\Temp\CrashLogs\os.txt
 
mkdir C:\Temp\CrashLogs\Dumps 2>nul
xcopy C:\Windows\Minidump C:\Temp\CrashLogs\Dumps\Minidump /E /I /Y 2>nul
xcopy C:\Windows\LiveKernelReports C:\Temp\CrashLogs\Dumps\LiveKernelReports /E /I /Y 2>nul
copy C:\Windows\MEMORY.DMP C:\Temp\CrashLogs\Dumps\ 2>nul
 
powershell -NoProfile -Command "Compress-Archive -Path 'C:\Temp\CrashLogs\*' -DestinationPath 'C:\Temp\CrashLogs.zip' -Force"
 
echo.
echo Done. Upload this file to the ticket:
echo C:\Temp\CrashLogs.zip
pause
```

Copy the CrashLogs.zip to your own laptop so you can investigate further.