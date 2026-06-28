# Clear Microsoft Teams Cache

Use this when Microsoft Teams is not loading correctly, showing stale data, or behaving unexpectedly.

> Note: This script closes Teams and clears the Teams cache for the currently signed-in Windows user. Teams may take longer than usual to open the first time after running this.

## Steps

1. Close Microsoft Teams if it is open.
2. Open **Command Prompt** as the affected user.
   - Administrator is usually not required.
   - Do not run this as a different admin account, otherwise it may clear the wrong user's Teams cache.
3. Paste the full script below into Command Prompt.
4. Wait for the confirmation popup.
5. Reopen Teams.

```
@echo off
echo Closing Teams...
taskkill /f /im ms-teams.exe >nul 2>&1
taskkill /f /im msteams.exe >nul 2>&1
taskkill /f /im Teams.exe >nul 2>&1
taskkill /f /im msedgewebview2.exe >nul 2>&1

echo Clearing New Teams cache...
rd /s /q "%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalCache" >nul 2>&1
rd /s /q "%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\TempState" >nul 2>&1
rd /s /q "%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\AC" >nul 2>&1
rd /s /q "%LOCALAPPDATA%\Packages\MSTeams_8wekyb3d8bbwe\LocalState" >nul 2>&1

echo Clearing Classic Teams cache...
rd /s /q "%APPDATA%\Microsoft\Teams\Application Cache" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\blob_storage" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\Cache" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\databases" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\GPUCache" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\IndexedDB" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\Local Storage" >nul 2>&1
rd /s /q "%APPDATA%\Microsoft\Teams\tmp" >nul 2>&1

echo.
echo Teams cache cleared. Please reopen Teams.

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Teams cache cleared. Please reopen Teams.','Teams cache cleared','OK','Information')"

exit

rem End of script
```