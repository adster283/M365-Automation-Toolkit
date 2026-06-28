# Sent Items for Shaired Mailboxes

### Check current "Sent Items" copy settings

Run the following command to confirm whether sent items are being copied to the shared mailbox:

```powershell
Get-Mailbox -Identity <SharedMailboxUPN> |
Select-Object -Property MessageCopy* |
Format-List
```

This will return the current status of:
MessageCopyForSentAsEnabled
MessageCopyForSendOnBehalfEnabled

<br>

### Ensure sent items are stored in the shared mailbox

If sent messages are not appearing in the shared mailbox Sent Items folder, enable both settings:

```
Set-Mailbox -Identity <SharedMailboxUPN> `
  -MessageCopyForSentAsEnabled $true `
  -MessageCopyForSendOnBehalfEnabled $true

```

This ensures:
Emails sent as the shared mailbox are saved in its Sent Items
Emails sent on behalf of the shared mailbox are also saved there