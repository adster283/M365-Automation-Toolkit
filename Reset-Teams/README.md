# Teams Reset Script

## Purpose

This folder contains user-device scripts for remediating common Microsoft Teams client issues by closing Teams and clearing local Teams cache data.

Use this when Teams is behaving strangely on a user's laptop and normal troubleshooting has not resolved the issue.

Common examples:

- Teams will not open correctly
- Teams is stuck loading or signing in
- Teams shows old profile/account data
- Chat, calendar, or meeting content is not refreshing
- Teams behaves differently for the same user on another device
- Add-ins, presence, or meeting join behaviour appears broken

## What the script does

The script is intended to run locally on the affected user's laptop. It closes Teams-related processes and clears local Teams cache folders.

Typical actions include:

- Closing classic Teams and new Teams processes
- Clearing Teams cache folders from the user's profile
- Removing temporary Teams web/cache data
- Allowing Teams to rebuild fresh local cache on next launch

## What the script does not do

This script does not:

- Delete the user's Microsoft 365 account
- Delete Teams messages, files, chats, channels, or meetings from Microsoft 365
- Remove Teams licensing
- Change Teams policies
- Fix tenant-side, licensing, service health, or conditional access issues
- Reinstall Teams unless the script is specifically extended to do so

## Requirements

- Windows user device
- Run as the affected signed-in user
- No Microsoft 365 admin role required
- No tenant access required
- Administrator rights are usually not required for user-profile cache cleanup

## Recommended usage

1. Ask the user to save any work in Teams.
2. Close Teams.
3. Run the script as the affected user.
4. Reopen Teams.
5. Allow Teams a few minutes to rebuild cache and resync.
6. Test the original issue again.

## When not to use this

Do not rely on this script if the issue is likely caused by:

- Microsoft 365 service health incidents
- Missing Teams license
- Conditional Access or sign-in blocks
- Disabled user account
- Teams policy restrictions
- Exchange calendar/mailbox issues affecting Teams calendar
- Device-wide network, proxy, VPN, or DNS problems

## Risk level

Low. This is a local cache reset. The main impact is that Teams may take longer to open the first time after the reset while it rebuilds local cache.
