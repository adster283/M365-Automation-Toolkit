# Collect Crash Logs

> **Important:** The collected logs may contain sensitive information. Review data before sharing and only use organisation-approved AI tools and services that meet your security and privacy requirements.

## Table of Contents

* [Purpose](#purpose)
* [Requirements](#requirements)
* [Runtime](#runtime)
* [Output](#output)
* [Recommended Usage](#recommended-usage)
* [What is Collected](#what-is-collected)
* [How to Collect Logs](#how-to-collect-logs)
* [Investigation Workflow](#investigation-workflow)

  * [Step 1 - Check Event Logs](#step-1---check-event-logs)
  * [Step 2 - Check for Dump Files](#step-2---check-for-dump-files)
  * [Step 3 - Use AI to Speed Up Analysis](#step-3---use-ai-to-speed-up-analysis)
  * [Step 4 - Review Supporting Data](#step-4---review-supporting-data)
  * [Step 5 - Form a Theory](#step-5---form-a-theory)
* [Ticket Notes](#ticket-notes)

## Purpose

Collects common Windows logs and diagnostic data used to investigate:

* Random restarts
* BSODs (Blue Screens)
* Sleep/wake issues
* Kernel-Power 41 events
* Driver crashes
* Performance-related crashes
* Unexpected shutdowns

## Requirements

* Administrator Command Prompt
* Windows 10 or Windows 11

## Runtime

Approximately 2-5 minutes.

## Output

The script creates:

```text
C:\Temp\CrashLogs.zip
```

## Recommended Usage

This script should be run on the affected user's device.

Once the script has completed:

1. Locate the generated ZIP file:

```text
C:\Temp\CrashLogs.zip
```

2. Copy the ZIP file to your own administrator workstation.

3. Perform all investigation and analysis steps from your workstation using tools such as:

* Event Viewer
* WinDbg Preview
* Microsoft Copilot
* ChatGPT

This keeps the investigation process separate from the user's device and allows you to use your own troubleshooting tools.

## What is Collected

### Event Logs

* System
* Application
* Diagnostics Performance
* Power Troubleshooter

### Power Information

* Supported sleep states
* Wake sources
* Wake timers
* Power requests
* Wake-capable devices
* Sleep Study report (if supported)

### System Information

* Windows version
* Last boot time
* Device manufacturer and model
* BIOS information
* Installed drivers
* Installed Windows updates

### Crash Data

* Minidumps
* Live Kernel Reports
* MEMORY.DMP (if available)

---

## How to Collect Logs

1. Open **Command Prompt as Administrator**
2. Open `SCRIPT.md`
3. Copy the entire script
4. Paste into Command Prompt
5. Wait for completion
6. Upload or copy:

```text
C:\Temp\CrashLogs.zip
```

to your administrator workstation.

---

## Investigation Workflow

After copying `CrashLogs.zip` to your administrator workstation and extracting the contents:

### Step 1 - Check Event Logs

Open:

```text
System.evtx
```

in Event Viewer.

Filter for:

```text
Critical
Error
```

Look for:

* Kernel-Power 41
* BugCheck events
* Driver failures
* Disk errors
* Unexpected shutdown events

If you find an obvious cause, investigate that first.

---

### Step 2 - Check for Dump Files

Open:

```text
Dumps\
```

Look for:

```text
MEMORY.DMP
*.dmp
```

If dump files exist:

1. Open them in WinDbg Preview
2. Run:

```text
!analyze -v
```

3. Review:

```text
BugCheck Code
Probably caused by
Faulting Driver
Failure Bucket
```

---

### Step 3 - Use AI to Speed Up Analysis

Paste the output of:

```text
!analyze -v
```

into Microsoft Copilot or ChatGPT.

Example:

```text
Please analyse this WinDbg output.

Explain the likely cause, whether it appears to be hardware, driver, firmware, Windows, power management, storage, or security software related.

Suggest next troubleshooting steps.
```

---

### Step 4 - Review Supporting Data

If the cause is still unclear, review:

```text
drivers.txt
windows-updates.txt
bios.txt
systeminfo.txt
sleepstudy.html
```

Common things to look for:

* Recent driver changes
* Recent Windows updates
* Outdated BIOS versions
* Sleep or wake failures
* Hardware-specific issues

---

### Step 5 - Form a Theory

Most crash investigations usually fall into one of these categories:

* Driver issue
* BIOS/Firmware issue
* Windows update issue
* Sleep/Modern Standby issue
* Storage issue
* Hardware issue
* Security software conflict

Avoid guessing. Document the evidence that supports your conclusion.

---

## Ticket Notes

Record:

* Device model
* Serial number
* Windows version
* BIOS version
* Crash timestamp
* BugCheck code (if present)
* Suspected cause
* Evidence found
* Actions taken

This information is often required if the issue needs to be escalated.
