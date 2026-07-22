# Troubleshoot Intune App Installs and Unexpected App Removals

> **Important:** Intune, Company Portal, event, and application inventory logs may contain usernames, tenant identifiers, device names, app IDs, internal paths, commands, and deployment configuration. Share the ZIP only through approved support systems.

## Purpose

Use this guide when:

- An Intune application does not install
- An app is stuck as pending, downloading, installing, failed, or not applicable
- Intune reports success but the app is missing or broken
- The installer succeeds but Intune reports failure
- An app unexpectedly disappears
- Intune may have uninstalled an app
- A newer app may have superseded and removed an older app
- A user may have selected Uninstall in Company Portal
- A Microsoft Store, AppX, MSIX, or MSI deployment fails
- The Intune Management Extension appears unhealthy

This folder contains:

- `README.md` — this troubleshooting knowledge base
- `Script.md` — the Intune-focused log collector

For Windows crashes, use the [crash troubleshooting guide](../Collect-Device-Crash-Logs/README.md).

When the ticket involves both stability and Intune deployment issues, use the [combined collector](../Collect-All-Device-Logs/Script.md).

---

# Scope

The Intune Management Extension logs are primarily used for **Win32 apps**, Intune PowerShell scripts, and related management activity.

The collector also exports supporting Windows logs for:

- MSI installation and removal
- Microsoft Store, AppX, and MSIX deployment
- Delivery Optimization and BITS
- Code Integrity
- AppLocker
- Microsoft Defender
- MDM policy and enrollment processing
- Company Portal activity

Microsoft 365 Apps created with the dedicated **Microsoft 365 Apps** app type are not installed by the Intune Management Extension. Their detailed installation flow may not appear in `AppWorkload.log`. Check the Microsoft 365 Apps deployment and Office installation logs separately.

---

# Quick Decision Tree

```text
Does the app appear in AppWorkload.log?
│
├─ No
│  ├─ Check assignment, exclusions, filters, licensing, and last check-in
│  ├─ Confirm the app type uses IME
│  ├─ Check IntuneManagementExtension.log and ClientHealth.log
│  └─ Confirm the device and user being investigated
│
└─ Yes
   │
   ├─ Not applicable
   │  └─ Check requirement rules and architecture
   │
   ├─ Already detected before install
   │  └─ Check for a false-positive detection rule
   │
   ├─ Download failed
   │  └─ Check content, disk, proxy, BITS, and Delivery Optimization
   │
   ├─ Install command failed
   │  └─ Check command, context, return code, timeout, and installer logs
   │
   ├─ Install returned success but app is not detected
   │  └─ Detection rule does not match the installed result
   │
   └─ Uninstall action appears
      └─ Check uninstall assignment, supersedence, Company Portal, or admin removal
```

---

# Information to Record Before Collection

Record the following in the ticket:

- App display name
- Intune app ID
- Device name
- Primary user and affected user
- Approximate incident time
- Intune error code and status details
- App type
- Required, Available, or Uninstall assignment
- User or System install behaviour
- Included groups
- Excluded groups
- Assignment filters
- Install command
- Uninstall command
- Detection rules
- Requirement rules
- Dependencies
- Supersedence relationships
- Whether **Allow available uninstall** is enabled
- Whether the user remembers selecting Uninstall
- Whether another technician or management tool may have removed it
- Last successful device check-in

Take screenshots or export configuration before changing the app. Otherwise the evidence can be lost during troubleshooting.

---

# Collecting the Logs

1. Open `Script.md`.
2. Copy the complete code block into Notepad.
3. Save it as:

   ```text
   Collect-Intune-App-Logs.cmd
   ```

4. Right-click the file and select **Run as administrator**.
5. Wait for the completion popup.
6. Upload or copy:

   ```text
   C:\Temp\IntuneAppLogs.zip
   ```

The script closes its elevated Command Prompt automatically on success or failure.

---

# What the Intune Collector Captures

## Intune Management Extension logs

```text
IME-Logs\
```

These are copied from:

```text
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs
```

## Company Portal logs

```text
CompanyPortal\<WindowsProfile>\
```

The collector checks each local Windows profile for Company Portal `Log_*.log` files.

This is useful when a user may have initiated an install or uninstall from Company Portal.

## Windows event logs

```text
Event-Logs\Application.evtx
Event-Logs\Intune-MDM-Admin.evtx
Event-Logs\AppX-Deployment.evtx
Event-Logs\AppX-Packaging.evtx
Event-Logs\BITS-Client.evtx
Event-Logs\Delivery-Optimization.evtx
Event-Logs\Code-Integrity.evtx
Event-Logs\Defender-Operational.evtx
Event-Logs\AppLocker-EXE-DLL.evtx
Event-Logs\AppLocker-MSI-Script.evtx
```

A channel may be missing or disabled on some devices. Check `Collection-Errors.txt`.

## Supporting state

```text
Installed-Apps-Machine.csv
Installed-Appx-All-Users.csv
IME-Service.txt
Intune-Scheduled-Tasks.csv
dsregcmd-status.txt
systeminfo.txt
disk-space.txt
winhttp-proxy.txt
ipconfig-all.txt
Collection-Summary.txt
```

These provide app inventory, IME state, join state, disk space, proxy configuration, network details, and collection time.

---

# Important IME Logs

## `AppWorkload.log`

Start here for Win32 app installations and removals.

It contains app deployment activity such as:

- Install and uninstall intent
- Content download
- Enforcement actions
- Install and uninstall command execution
- Exit codes
- Detection results
- Dependencies
- Supersedence
- Retry activity

## `AppActionProcessor.log`

Use this for:

- Detection-rule processing
- Requirement and applicability checks
- Whether an app is detected
- Whether the device is eligible for the app

## `IntuneManagementExtension.log`

Use this for:

- IME check-ins
- Policy requests
- Policy processing
- Status reporting
- Service and communication problems
- Whether app policy reached the device

## `ClientHealth.log`

Use this when:

- IME appears unhealthy
- The service repeatedly repairs or restarts
- App processing does not occur
- Policy is not being handled normally

## `AgentExecutor.log`

Use this for:

- Intune PowerShell scripts
- Script-based requirements
- Script-based detection
- Remediations or scripts related to the app

## Rotated logs

Older logs can be prefixed or numbered depending on IME version and rotation.

Search every copy when the incident was not recent.

---

# Reading the Logs Efficiently

Use CMTrace when available.

## Search by app ID first

IME logs often use the Intune app GUID instead of the friendly display name.

Search for:

```text
<App GUID>
```

Then follow entries around the incident time.

## Search terms

```text
Install
Uninstall
Detection
Detected
NotDetected
Not detected
Applicability
Not applicable
Requirement
Enforcement
Download
Content
Dependency
Supersedence
Exit code
Return code
Retry
Failed
Error
0x
```

## Correlate the full sequence

For an install, look for:

```text
Assignment received
→ Requirement evaluation
→ Detection before install
→ Content download
→ Install command
→ Exit code
→ Detection after install
→ Status reported
```

For an uninstall, look for:

```text
Uninstall intent or action
→ Uninstall command
→ Exit code
→ Detection after uninstall
→ Not detected
→ Status reported
```

A single `Not detected` line does not prove Intune removed the app.

---

# Portal Checks Before Deep Log Analysis

In Intune admin center:

1. Go to **Apps > All apps**.
2. Select the app.
3. Review **Device install status**.
4. Record:
   - Status
   - Status details
   - Last check-in
   - User
   - Device
5. Review **Properties**:
   - Program
   - Requirements
   - Detection rules
   - Dependencies
   - Supersedence
   - Assignments
6. Check **Troubleshooting + support** for the affected user.
7. Use **Collect logs** from the app installation details when available.

Do not rely only on the portal status. Portal reporting can lag behind local activity.

---

# Why Did the App Not Install?

## 1. The app never appears in IME logs

Likely areas:

- Wrong user or device targeted
- Device or user missing from the included group
- Excluded group applies
- Assignment filter excludes the device
- App is Available and the user never selected Install
- Device has not checked in
- User-based assignment has not processed for the affected user
- IME is missing or unhealthy
- The app type is not handled by IME
- The wrong device logs were collected

Check:

```text
IntuneManagementExtension.log
ClientHealth.log
IME-Service.txt
Intune-Scheduled-Tasks.csv
dsregcmd-status.txt
```

Then verify the app's assignments and the device's last check-in in Intune.

### Assignment filter warning

Available apps do not always appear in the same device status reporting as Required apps. Use the assignment filter evaluation reports when a filter may be excluding the device.

---

## 2. The app is reported as not applicable

Search for:

```text
Applicability
Not applicable
Requirement
```

Review:

- Minimum Windows version
- Architecture
- Disk-space requirement
- Memory requirement
- File requirement
- Registry requirement
- Script requirement
- 32-bit versus 64-bit setting

A failed requirement means Intune received the assignment but decided not to install the app on that device.

For a custom requirement script, confirm:

- It runs correctly as the configured context
- Its output matches the expected data type
- It returns the correct exit code
- It does not depend on a user profile when running as System

---

## 3. Detection says the app is already installed

Search for:

```text
Detection
Detected
Detection state
```

Possible false positives:

- A leftover file
- A leftover registry value
- An older version satisfies the rule
- Another edition uses the same path
- The rule checks the wrong architecture
- A custom detection script always returns success
- Version comparison is too broad
- An MSI product code points to a different package state

If detection is true before enforcement, Intune will normally skip the install.

Compare the detection rule with:

```text
Installed-Apps-Machine.csv
Installed-Appx-All-Users.csv
```

Also inspect the actual file and registry path on the device.

---

## 4. Content download failed

Search `AppWorkload.log` for:

```text
Download
Content
Cache
Delivery Optimization
DO
BITS
```

Review:

```text
Event-Logs\BITS-Client.evtx
Event-Logs\Delivery-Optimization.evtx
disk-space.txt
winhttp-proxy.txt
ipconfig-all.txt
```

Possible causes:

- Proxy or firewall restrictions
- Delivery Optimization failure
- BITS failure
- Insufficient free disk space
- Device lost connectivity
- Content package problem
- Security software blocked or removed content
- Stale cached content

Do not collect or manually alter the IME content cache unless there is a clear reason and an approved procedure.

---

## 5. The install command did not work

Compare the command in `AppWorkload.log` with the app's **Program** settings.

Check:

- Executable name
- Relative path
- Quotes around paths
- Silent switches
- Working directory assumptions
- Installer file included in the `.intunewin` package
- User versus System context
- 32-bit versus 64-bit behaviour
- Interactive prompts
- Child processes
- Mapped-drive dependencies
- Network-share dependencies
- Reboot behaviour

A command that works while signed in as an administrator may still fail when IME runs it as `SYSTEM`.

Test the exact command in the same context used by Intune.

---

## 6. The installer returned an error code

Search for:

```text
Exit code
Return code
Result
```

Compare the returned value with:

- Vendor installer documentation
- MSI error documentation
- The return-code mappings configured in Intune

Intune return-code handling can classify a code as:

- Success
- Soft reboot
- Hard reboot
- Retry
- Failed

Check `Application.evtx` for `MsiInstaller` entries when the package uses MSI.

---

## 7. The installation timed out

Review **Installation time required** in the app's Program settings.

For Win32 apps, Intune uses the configured timeout, with 60 minutes as the default and 1,440 minutes as the maximum.

Possible causes:

- Installer waits for user input
- Installer hangs
- Another MSI installation is active
- A child process continues after the parent exits
- Reboot is pending
- Security software delays the installer
- The configured timeout is too short

A timeout does not necessarily stop the installer process immediately. Confirm the local state before retrying.

---

## 8. Installer returned success, but Intune reports failure

This usually means post-install detection failed.

Typical sequence:

```text
Install command runs
→ Exit code is successful
→ Detection runs
→ App is not detected
→ Intune reports failure
```

Check:

- Installed path differs from the detection path
- Per-user install was used but detection expects a machine install
- 32-bit app writes to the 32-bit registry view
- File version differs from the expected version
- MSI product code changed
- Custom detection script output or exit code is wrong
- A reboot is required before detection becomes valid
- Security software removed the installed file
- The installer launched asynchronously and exited too early

Detection should confirm the final installed state, not merely that an installer ran.

---

## 9. Dependency failure

Search for:

```text
Dependency
Dependent
```

Check:

- Which dependency failed
- Whether dependency detection is correct
- Dependency install order
- Requirement rules on each dependency
- Whether a dependency was changed or deleted
- Whether the dependency is itself superseded
- Whether the Company Portal uninstall option is hidden because dependencies exist

Troubleshoot the first failing dependency before the parent app.

---

## 10. Supersedence problem

Search for:

```text
Supersedence
Superseded
Uninstall previous
```

Check whether:

- The correct app supersedes the old app
- **Uninstall previous version** is enabled
- Old and new detection rules overlap
- The replacement app failed after the old app was removed
- A supersedence chain contains unintended relationships
- The new installer already performs an in-place update and should not separately uninstall the old version

If Intune removes the old app and the replacement then fails, the user can be left with neither version.

---

## 11. AppLocker, Code Integrity, or Defender blocked the app

Review:

```text
Event-Logs\AppLocker-EXE-DLL.evtx
Event-Logs\AppLocker-MSI-Script.evtx
Event-Logs\Code-Integrity.evtx
Event-Logs\Defender-Operational.evtx
```

Look for events at the exact install time involving:

- Installer executable
- MSI
- PowerShell or script
- DLL
- Temporary extraction path
- Quarantine or remediation
- WDAC or App Control policy
- AppLocker policy

An Intune exit code may only show that the process failed. These security logs can explain why.

---

## 12. Store, AppX, or MSIX install failed

Review:

```text
Event-Logs\AppX-Deployment.evtx
Event-Logs\AppX-Packaging.evtx
Installed-Appx-All-Users.csv
```

Check for:

- Package dependency failure
- Architecture mismatch
- Signing or certificate issue
- Registration failure
- Existing package conflict
- User-specific package state
- Microsoft Store service or licensing issue

IME logs may not contain the full package-deployment failure for every Store or AppX scenario.

---

## 13. Microsoft 365 Apps deployment failed

When the app was created using the dedicated Microsoft 365 Apps app type:

- Do not expect the complete flow in `AppWorkload.log`
- Check the Microsoft 365 Apps deployment configuration
- Check Office Click-to-Run and Office setup logs
- Confirm no Office application is blocking the deployment
- Check for existing MSI-based Office products
- Confirm architecture and update-channel configuration
- Confirm network access to Microsoft 365 content endpoints

When deeper IME-style control is needed during Autopilot, some organisations package Microsoft 365 Apps as Win32, but this should follow an approved deployment design.

---

# Why Was the App Removed?

## 1. Look for explicit uninstall evidence

Search:

```text
Uninstall
Uninstall command
Enforcement
Intent
Action
Exit code
Not detected
```

Strong evidence that Intune initiated removal is a sequence showing:

```text
Uninstall action
→ Configured uninstall command
→ Process execution
→ Exit code
→ Detection confirms not installed
```

Record the app ID, timestamp, command, and exit code.

---

## 2. Check for an Uninstall assignment

Review the app's assignments for:

- A direct Uninstall group
- A nested or dynamic group change
- A user or device recently added to an uninstall group
- An exclusion changing effective targeting
- An assignment filter change
- Conflicting user and device intent

Intune resolves conflicting assignment intents according to assignment precedence rules. Do not assume the visible assignment you first notice is the effective intent.

Export or screenshot all assignments before changing them.

---

## 3. Check supersedence

A superseding Win32 app can remove an older app when **Uninstall previous version** is enabled.

Confirm:

- Which app superseded the removed app
- When the relationship was created or changed
- Whether the old uninstall command ran
- Whether the replacement installed
- Whether replacement detection failed
- Whether the new installer already handled the upgrade

This is a common explanation when an old app disappears and the expected new version is also missing.

---

## 4. Check Company Portal user uninstall

A user can uninstall some Win32 and Microsoft Store apps through Company Portal.

For Win32 apps, check **Allow available uninstall**.

Review:

```text
CompanyPortal\<AffectedUser>\
AppWorkload.log
```

Look for user-initiated activity at the removal time.

An available app with dependencies may not show the uninstall option even when the setting is enabled.

---

## 5. Check the Intune Remove apps and configuration device action

An Intune administrator can use the interactive **Remove apps and configuration** device action.

Review:

- Device action history
- Audit logs
- Ticket or change records
- Local uninstall timestamp

Intune can later reapply assigned items to return the device to its intended state, so a removed Required app may return.

---

## 6. Check MSI removal evidence

Open:

```text
Event-Logs\Application.evtx
```

Filter by source:

```text
MsiInstaller
```

Look for:

- Product name
- Product code
- Removal or upgrade
- MSI error code
- Timestamp

MSI events can prove that Windows Installer removed or upgraded a product, but they may not identify who requested it.

---

## 7. Check Store, AppX, or MSIX removal evidence

Review:

```text
Event-Logs\AppX-Deployment.evtx
Event-Logs\AppX-Packaging.evtx
Installed-Appx-All-Users.csv
```

Look for:

- Package removal
- Deregistration
- Package replacement
- User-specific removal
- Failed update that changed package state

---

## 8. Check security and remediation tools

Possible removers include:

- Microsoft Defender remediation
- Third-party endpoint security
- AppLocker or App Control policy
- Intune Remediations
- Intune PowerShell scripts
- Another RMM or software-management platform
- Vendor updaters
- Cleanup tools
- A technician

Review Defender and Code Integrity logs, `AgentExecutor.log`, scheduled tasks, and management change records.

---

## 9. Determine whether the app was actually removed

`Not detected` does not automatically mean uninstalled.

Detection can change because:

- The detection rule was edited
- A self-update changed the executable version
- The app moved paths
- A registry value changed
- The MSI product code changed
- A per-user app exists under another profile
- A custom detection script failed
- An old leftover used to satisfy detection and was cleaned up

Confirm:

- Installed Apps
- Executable path
- Services
- Registry uninstall entries
- App launch
- `Installed-Apps-Machine.csv`
- `Installed-Appx-All-Users.csv`

---

## 10. Required versus Available behaviour

### Required

A Required assignment represents the desired state that the app should be installed.

If the app is removed or becomes not detected, Intune can attempt enforcement again during later processing.

Repeated reinstall cycles usually indicate:

- Another tool keeps removing the app
- Detection is unstable
- Install succeeds only temporarily
- Conflicting assignment intent
- Supersedence or replacement behaviour

### Available

An Available app is user-initiated.

If the user removes it, it can remain absent until the user selects Install again, depending on app type and configuration.

Do not assume an Available app will be automatically reinstalled like a Required app.

---

# Evidence Matrix

| Evidence | Likely interpretation |
|---|---|
| App never appears in IME logs | Targeting, app type, check-in, wrong device, or IME issue |
| Requirement reports not applicable | Device does not satisfy requirement rules |
| Detected before install | Detection false positive or app already present |
| Content download error | Network, proxy, DO, BITS, disk, content, or security issue |
| Install command returns failure | Installer, command line, context, or vendor failure |
| Install returns success then detection fails | Detection does not match final installed state |
| Explicit uninstall command and exit code | Intune or IME initiated uninstall enforcement |
| Supersedence plus old-app uninstall | App replacement or update configuration |
| Company Portal activity at removal time | User may have initiated uninstall |
| MSI removal event without IME uninstall | Manual, vendor, admin, or another management tool may have removed it |
| Not detected without uninstall execution | Detection changed or app was removed outside Intune |
| Defender quarantine at install time | Security remediation blocked or removed content |
| AppLocker or Code Integrity block | Policy prevented installer, script, DLL, or app execution |
| Required app repeatedly returns | Intune is re-enforcing desired state |
| Available app remains removed | Often expected until user requests installation again |

---

# Common Patterns

## Pattern A — Intune says installed, user says missing

Check:

1. Detection rule
2. Whether it is installed for another user
3. 32-bit versus 64-bit path
4. Inventory CSV files
5. App launch path
6. Whether a shortcut is missing but the app exists

## Pattern B — Intune says failed, app is installed

Check:

1. Installer exit code
2. Return-code mapping
3. Post-install detection
4. Reboot requirement
5. Asynchronous installer behaviour
6. Version comparison

## Pattern C — Old version removed, new version missing

Check:

1. Supersedence
2. **Uninstall previous version**
3. Old uninstall success
4. New download/install failure
5. New detection failure
6. Requirement rule for the replacement

## Pattern D — App repeatedly installs and disappears

Check:

1. Required and Uninstall assignment conflicts
2. Remediation scripts
3. Security products
4. Vendor updater
5. Detection instability
6. Supersedence
7. Another RMM or package manager

## Pattern E — Nothing happens after assignment

Check:

1. Last check-in
2. Group membership
3. Exclusions and filters
4. App type
5. IME service
6. IME check-in and policy request logs
7. User versus device targeting
8. Available app requiring user action

---

# Escalation Checklist

Provide:

- `IntuneAppLogs.zip`
- App name and Intune app ID
- Device and affected user
- Incident timestamp
- Portal status and status details
- Last check-in
- App type
- Install and uninstall commands
- Install behaviour
- Return-code mappings
- Timeout
- Requirement rules
- Detection rules
- Assignments, exclusions, and filters
- Dependencies
- Supersedence configuration
- **Allow available uninstall** setting
- Relevant IME log sequence
- Exit or error code
- Matching Windows event
- Whether Company Portal activity exists
- Whether another management or security tool is present
- Changes made and retest result

---

# Limitations

- IME logs rotate and older evidence can be overwritten.
- The collector is a point-in-time snapshot.
- Portal reporting can lag.
- A `Not detected` result does not prove Intune uninstalled the app.
- Local logs may prove what ran but not always which administrator changed the cloud assignment.
- Audit logs and change records may be needed for attribution.
- Company Portal logs may be unavailable if the profile was removed or the app was reset.
- Some event channels are disabled or absent on some Windows editions.
- The collector does not copy the IME content cache or installer packages.
- Dedicated Microsoft 365 Apps deployments require additional Office-specific troubleshooting.

---

# Microsoft References

- [Intune Management Extension for Windows](https://learn.microsoft.com/en-us/intune/device-management/tools/management-extension-windows)
- [Troubleshoot Win32 app issues](https://learn.microsoft.com/en-us/intune/app-management/deployment/troubleshoot-win32)
- [Add and assign Win32 apps](https://learn.microsoft.com/en-us/intune/app-management/deployment/add-win32)
- [Win32 app management](https://learn.microsoft.com/en-us/intune/app-management/deployment/win32)
- [Assign apps to groups](https://learn.microsoft.com/en-us/intune/app-management/deployment/assign-groups)
- [Add Win32 app supersedence](https://learn.microsoft.com/en-us/intune/app-management/deployment/configure-win32-supersedence)
- [Monitor app information and assignments](https://learn.microsoft.com/en-us/intune/app-management/monitor-assignments)
- [Intune reports](https://learn.microsoft.com/en-us/intune/device-management/reports/overview)
- [Install and uninstall apps from Company Portal](https://learn.microsoft.com/en-us/intune/user-help/apps/install-apps-windows)
- [Collect Company Portal logs manually](https://learn.microsoft.com/en-us/intune/user-help/diagnostics/collect-logs-company-portal-windows)
- [Remove apps and configuration device action](https://learn.microsoft.com/en-us/intune/device-management/actions/remove-apps-config)
- [Troubleshoot assignment filters](https://learn.microsoft.com/en-us/intune/fundamentals/filters/troubleshoot)
- [Add Microsoft 365 Apps to Windows devices](https://learn.microsoft.com/en-us/intune/app-management/deployment/add-microsoft-365-windows)
