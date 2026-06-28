# Reset Outlook Profile and Cache

Use this script when Outlook is failing to open, stuck loading, repeatedly prompting for sign-in, or has a corrupted profile/cache.

> Important: This script resets the Outlook profile for the currently signed-in Windows user. It deletes local OST cache files and removes the local Outlook profile registry key. Mail stored in Microsoft 365/Exchange should resync after the profile is recreated, but local-only data may not. Only run this when a profile reset is appropriate.

> A backup of the existing Outlook profile registry key is exported to the user's Desktop before the profile is deleted.

## Steps

1. Close Outlook if it is open.
2. Open **Command Prompt** as the affected user.

   * Administrator is usually not required.
   * Do not run this as a different admin account, otherwise it may reset the wrong user's Outlook profile.
3. Paste the full script below into Command Prompt.
4. Wait for the confirmation popup.
5. Create a new Outlook profile when prompted.
6. Reopen Outlook and allow the mailbox to resync.

## Script

```bat
@echo off
echo Closing Outlook...

taskkill /f /im outlook.exe >nul 2>&1
taskkill /f /im newoutlook.exe >nul 2>&1

echo Clearing Outlook cache and OST files...

rem Clear OST files
del /q "%LOCALAPPDATA%\Microsoft\Outlook\*.ost" >nul 2>&1

rem Clear temp Outlook cache
rd /s /q "%LOCALAPPDATA%\Microsoft\Outlook\RoamCache" >nul 2>&1
rd /s /q "%LOCALAPPDATA%\Microsoft\Office\16.0\Wef" >nul 2>&1

rem Clear autocomplete cache
del /q "%LOCALAPPDATA%\Microsoft\Outlook\*.dat" >nul 2>&1

echo Backing up existing Outlook profile registry key...

reg export "HKCU\Software\Microsoft\Office\16.0\Outlook\Profiles" "%USERPROFILE%\Desktop\OutlookProfileBackup.reg" /y >nul 2>&1

echo Resetting Outlook profiles...

reg delete "HKCU\Software\Microsoft\Office\16.0\Outlook\Profiles" /f >nul 2>&1

echo Launching Mail profile setup...

control mlcfg32.cpl

echo.
echo Outlook reset complete.
echo Please create a new profile and reopen Outlook.

start "" powershell.exe -NoProfile -WindowStyle Hidden -Command "Add-Type -AssemblyName PresentationFramework; [void][System.Windows.MessageBox]::Show('Outlook reset complete. Please create a new profile and reopen Outlook. A backup was saved to the Desktop as OutlookProfileBackup.reg.','Outlook reset complete','OK','Information')"

exit

rem End of script
```
