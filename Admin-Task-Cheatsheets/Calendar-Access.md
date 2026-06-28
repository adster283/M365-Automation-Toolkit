# Calendar Cheatsheet

### Install required Modules

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
```

```powershell
Install-Module -Name ExchangeOnlineManagement
```

<br>

### Connecting (With named account)

```powershell
connect-exchangeonline
```

Connecting (With GDAP)

```cmd
connect-exchangeonline -delegatedorganization example.com
```

<br>

### Check permissions

```powershell
Get-MailboxFolderPermission -identity emailaddress:\calendar
```

<br>

### Give access (user does not have access)

```powershell
add-MailboxFolderPermission -identity emailaddress:\calendar -user emailaddress -accessrights editor
```

Change access (user does have access)

```powershell
set-MailboxFolderPermission -identity emailaddress:\calendar -user emailaddress -accessrights owner
```

The main permission you may need to set:

- Owner \= admin of the calendar
- Editor \= full access (most common)
- Reviewer \= read-only
- AvailabilityOnly / LimitedDetails \= free/busy sharing

<br>

### Remove access

```powershell
Remove-MailboxFolderPermission -identity emailaddress:\calendar -user emailaddress
```