# Outlook Reset / Cache Cleanup Script

## Purpose

This folder contains user-device scripts for Outlook troubleshooting. These scripts are intended for cases where Outlook has local client/cache problems and basic troubleshooting has not resolved the issue.

Common examples:

- Outlook is stuck loading
- Outlook search or autocomplete behaves incorrectly
- Outlook repeatedly prompts or fails to open the mailbox
- Old profile/cache data appears to be causing issues
- The same mailbox works in Outlook on the web but not on the laptop

## Important warning

Use this script carefully. Outlook remediation is more than just clearing cache.

Unlike Teams cache cleanup, Outlook can involve local mailbox cache files, profiles, add-ins, autocomplete data, OST rebuilds, account configuration, and sometimes user-specific mailbox/profile issues. A heavy Outlook reset can force the user to recreate their Outlook profile and resync their mailbox.

For most Outlook issues, prefer manual remediation first.

Recommended manual checks before using this script:

1. Confirm Outlook on the web works for the user.
2. Confirm Microsoft 365 service health is clean.
3. Restart Outlook and the laptop.
4. Start Outlook in safe mode: `outlook.exe /safe`.
5. Disable suspicious COM add-ins.
6. Check mailbox size and sync status.
7. Check credential prompts and account sign-in state.
8. Create a new Outlook profile manually if needed.
9. Only then consider cache/profile cleanup.

## What the script may do

Depending on the script version, it may:

- Close Outlook and new Outlook processes
- Delete local OST cache files
- Clear Outlook RoamCache
- Clear Office web add-in cache
- Clear autocomplete/cache files
- Export existing Outlook profile registry keys as a backup
- Delete local Outlook profile registry keys to force profile recreation

## What the script does not do

This script does not:

- Delete emails from Exchange Online
- Delete mailbox data from Microsoft 365
- Remove the user's mailbox license
- Fix Exchange Online service issues
- Fix mailbox corruption in the cloud
- Fix Conditional Access, MFA, or sign-in policy issues
- Guarantee Outlook will reopen without profile reconfiguration

## Requirements

- Windows user device
- Run as the affected signed-in user
- Outlook must be closed or allowed to be closed by the script
- No Microsoft 365 admin role required for local cache cleanup
- Administrator rights may not be required for user-profile cleanup, but some environments may restrict registry or profile changes

## Recommended usage

1. Confirm the user can access email in Outlook on the web.
2. Confirm whether the user uses classic Outlook or new Outlook.
3. Ask the user to close Outlook and save any work.
4. Run the script as the affected user.
5. Reopen Outlook.
6. Recreate or select an Outlook profile if prompted.
7. Allow time for the mailbox to resync.
8. Retest the original issue.

## Expected impact

After running a full reset, Outlook may need to redownload mailbox data. This can take time depending on:

- Mailbox size
- Cached Exchange Mode settings
- Network speed
- Shared mailbox count
- Online Archive usage

The user may temporarily see missing recent email, incomplete search results, or folders still syncing while the local cache rebuilds.

## Risk level

Medium. This script can force profile recreation and mailbox resync. Use manual Outlook remediation first unless the issue is clearly local cache/profile corruption.
