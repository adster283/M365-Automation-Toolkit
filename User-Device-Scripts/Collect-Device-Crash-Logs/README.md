# Collect Device Crash Logs

> **Important:** Crash dumps and event logs may contain usernames, device names, file paths, application data, and fragments of memory. Store and share the ZIP only through approved support systems.

## Purpose

Use this collector to investigate Windows device stability problems such as:

- Blue screens and bug checks
- Random restarts
- Unexpected shutdowns
- Freezes followed by a forced restart
- Kernel-Power Event ID 41
- Driver or firmware crashes
- Sleep, wake, and Modern Standby problems
- Live kernel events
- Suspected storage, power, or hardware instability

This folder contains:

- `README.md` — the troubleshooting knowledge base
- `Script.md` — the crash-only collection script

For Intune application problems, use the dedicated [Intune app troubleshooting guide](../Collect-Intune-App-Logs/README.md).

When a ticket involves both Windows instability and Intune application behaviour, use the [combined collector](../Collect-All-Device-Logs/Script.md).

---

## Requirements

- Windows 10 or Windows 11
- Local administrator rights
- Enough free space under `C:\Temp` for any available dump files
- The script must be saved and run as a `.cmd` or `.bat` file

Do not paste the script line-by-line into an existing Command Prompt. Batch labels, variables, and control flow are designed to run from a saved script file.

---

## Output

The crash-only collector creates:

```text
C:\Temp\CrashLogs.zip
```

The temporary working folder is:

```text
C:\Temp\CrashLogs
```

Running the script again replaces the previous working folder and ZIP.

Large `MEMORY.DMP` files can make the ZIP several gigabytes and can increase collection time.

---

## What Is Collected

### Windows event logs

```text
System.evtx
Application.evtx
Diagnostics-Performance.evtx
Power-Troubleshooter.evtx
```

These help identify:

- Bug checks and unexpected shutdowns
- Driver, service, disk, storage, and hardware events
- Application crashes
- Startup and shutdown performance problems
- Sleep and wake activity

### Crash dumps

```text
Dumps\Minidump
Dumps\LiveKernelReports
Dumps\MEMORY.DMP
```

The script copies these only when they exist.

### System information

```text
systeminfo.txt
drivers.txt
windows-updates.txt
bios.txt
model.txt
os.txt
```

These provide the Windows build, update history, BIOS information, model, serial number, drivers, and last boot time.

### Power information

```text
powercfg-a.txt
lastwake.txt
waketimers.txt
power-requests.txt
wake-armed-devices.txt
sleepstudy.html
```

Some devices do not support Sleep Study. A failed Sleep Study export does not mean the whole collection failed.

### Collection errors

```text
Collection-Errors.txt
```

Review this when an expected file is missing. Some event channels and dump paths do not exist on every device.

---

# Collecting the Logs

1. Open `Script.md`.
2. Copy the complete code block into Notepad.
3. Save it as:

   ```text
   Collect-Crash-Logs.cmd
   ```

4. Right-click the file and select **Run as administrator**.
5. Wait for the completion popup.
6. Upload or copy:

   ```text
   C:\Temp\CrashLogs.zip
   ```

7. Perform the investigation from an administrator workstation when possible.

The script closes its elevated Command Prompt automatically on success or failure so an unused administrator shell is not left open on the user's device.

---

# Crash Investigation Workflow

## Step 1 — Establish the incident time

Before opening the logs, record:

- Approximate crash or restart time
- What the user was doing
- Whether the screen turned blue, black, or froze
- Whether the device restarted by itself
- Whether power was lost
- Whether the issue happened while docking, undocking, sleeping, waking, or using a specific application
- Whether the issue repeats under a specific workload

Time correlation is critical. Avoid treating unrelated warnings from hours or days earlier as the cause.

---

## Step 2 — Start with `System.evtx`

Open `System.evtx` in Event Viewer and inspect the period immediately before and after the incident.

Useful event sources include:

```text
Kernel-Power
BugCheck
WHEA-Logger
Disk
Ntfs
stornvme
storahci
storport
volmgr
Display
Kernel-PnP
Service Control Manager
Power-Troubleshooter
```

### Kernel-Power Event ID 41

Event ID 41 means Windows detected that the previous shutdown was not clean.

It does **not** identify the root cause by itself.

Possible causes include:

- A blue screen
- Sudden power loss
- A forced power-button shutdown
- A system freeze followed by a reset
- Hardware or firmware failure
- Battery, dock, charger, or power-delivery problems

Check the event details for a bug-check code, but correlate it with a dump file and nearby events before reaching a conclusion.

### BugCheck evidence

If a bug check occurred, look for:

- A `BugCheck` event
- A bug-check code recorded in Kernel-Power Event ID 41
- Event ID 1001 entries
- A matching `.dmp` file

A bug-check event plus a matching dump is much stronger evidence than Event ID 41 alone.

### Hardware evidence

Repeated `WHEA-Logger` events near the crash can indicate a hardware or firmware-reported error. Correlate them with the affected component, BIOS state, drivers, and vendor diagnostics.

### Storage evidence

Repeated disk, controller, NVMe, NTFS, reset, timeout, or bad-block events can point toward:

- Storage-device failure
- Controller or firmware issues
- Cabling or docking problems
- Driver problems
- File-system corruption

Do not assume every isolated storage warning caused the crash. Look for repetition and timestamp alignment.

---

## Step 3 — Check `Application.evtx`

Use `Application.evtx` when:

- A specific program crashed but Windows stayed running
- The device froze while using one application
- A service repeatedly failed
- Security software, an updater, or another background process may be involved

Look for:

```text
Application Error
Windows Error Reporting
.NET Runtime
Application Hang
SideBySide
MsiInstaller
```

Record the faulting application, faulting module, exception code, and timestamp.

An application crash is not automatically the cause of a Windows restart. Confirm whether it occurred before the system failure or merely after startup.

---

## Step 4 — Check for dump files

Look under:

```text
Dumps\
```

Prefer the dump whose timestamp matches the incident.

Typical locations in the ZIP include:

```text
Dumps\Minidump\*.dmp
Dumps\LiveKernelReports\*.dmp
Dumps\MEMORY.DMP
```

### Analyze with WinDbg

1. Open the dump in WinDbg.
2. Allow symbols to load.
3. Run:

   ```text
   !analyze -v
   ```

4. Record:

   ```text
   BugCheck
   Arguments
   MODULE_NAME
   IMAGE_NAME
   FAILURE_BUCKET_ID
   PROCESS_NAME
   STACK_TEXT
   ```

### Interpret carefully

`Probably caused by` is a lead, not always proof.

A Microsoft system module may appear because it detected the failure, even when the underlying cause is:

- A third-party driver
- Memory corruption
- Firmware
- Storage
- Hardware
- Security software
- A dock or peripheral

Look for consistency across multiple dumps. Several dumps naming the same third-party driver are stronger evidence than one isolated result.

---

## Step 5 — If no dump exists

No dump can mean:

- The device lost power before Windows could write one
- The system froze instead of bug checking
- Dump creation is disabled or misconfigured
- The page file was unavailable or too small
- Storage failed during dump creation
- The dump was cleaned up
- The issue was an application crash rather than a Windows crash

Check:

```text
System.evtx
Application.evtx
Collection-Errors.txt
powercfg files
sleepstudy.html
```

Also review the device's configured startup and recovery dump settings when repeated incidents produce no dump.

---

## Step 6 — Review sleep and power evidence

Use:

```text
sleepstudy.html
lastwake.txt
waketimers.txt
power-requests.txt
wake-armed-devices.txt
powercfg-a.txt
```

These are most useful for:

- Battery drain during sleep
- Unexpected waking
- Failure to enter sleep
- Modern Standby issues
- A device or process preventing sleep
- Docking and power-state transitions

`lastwake.txt`, wake timers, and active power requests show the state at collection time. They may not represent the state at the exact time of an older incident.

---

## Step 7 — Correlate drivers, updates, BIOS, and model

Review:

```text
drivers.txt
windows-updates.txt
bios.txt
model.txt
os.txt
systeminfo.txt
```

Check for:

- A recent Windows or driver update matching the first incident
- An outdated BIOS or firmware package
- A known device-model issue
- Very old graphics, storage, chipset, network, or dock drivers
- Repeated failures after a hardware or peripheral change

Use the device manufacturer's supported update tools and documentation before installing random third-party driver packages.

---

# Symptom-to-Evidence Guide

| Symptom | Start with | Stronger evidence |
|---|---|---|
| Blue screen | Dump files and `System.evtx` | Matching bug-check event and dump |
| Instant restart or power-off | `System.evtx` around Event ID 41 | Power evidence, WHEA, bug-check data, hardware diagnostics |
| Black screen then recovery | LiveKernelReports and display events | Matching graphics/watchdog dump |
| Freeze requiring power button | Events immediately before Event ID 41 | Repeated driver, storage, WHEA, or application-hang pattern |
| Sleep or wake failure | Sleep Study and power logs | Repeated state transition or device pattern |
| One app closes | `Application.evtx` | Repeated faulting module and exception |
| Crash after update | Updates, drivers, event timestamps | Reproducible timing and rollback/update result |
| Different drivers named in every dump | Multiple dumps and hardware evidence | Memory, storage, firmware, or broad corruption testing |

---

# Forming a Conclusion

Most investigations fall into one or more of these categories:

- Third-party driver
- BIOS or firmware
- Windows update
- Graphics subsystem
- Storage subsystem
- Memory or CPU hardware
- Sleep or power management
- Dock, charger, battery, or peripheral
- Security software
- Application-only failure
- Insufficient evidence

Document the evidence supporting the conclusion.

Avoid statements such as “Kernel-Power 41 means the power supply is faulty.” Event ID 41 records an unclean shutdown; it does not prove a specific component failed.

---

# Escalation Checklist

Include:

- Device name
- Manufacturer and model
- Serial number
- Windows edition, version, and build
- BIOS version
- Incident date and time
- User-visible symptom
- Frequency and reproduction steps
- Bug-check code
- Dump filename
- WinDbg `!analyze -v` output
- Faulting driver or module
- Relevant event sources and IDs
- Recent updates or device changes
- Dock, charger, and peripheral details
- Troubleshooting already completed
- Whether vendor hardware diagnostics passed

---

# Limitations

- The collector is a point-in-time snapshot.
- Old event entries and dumps may have been overwritten or cleaned up.
- A memory dump can contain sensitive information.
- Event ID 41 does not identify the root cause by itself.
- `Probably caused by` in WinDbg should not be treated as final proof without correlation.
- The script does not run hardware diagnostics.
- The script does not modify crash-dump settings.
- Sleep Study is not supported on every device.

---

# Microsoft References

- [Event ID 41: The system has rebooted without cleanly shutting down first](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/event-id-41-restart)
- [Install WinDbg](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/)
- [Generate a kernel or complete crash dump](https://learn.microsoft.com/en-us/troubleshoot/windows-client/performance/generate-a-kernel-or-complete-crash-dump)
