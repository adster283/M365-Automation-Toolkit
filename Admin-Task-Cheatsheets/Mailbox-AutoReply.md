# Set Auto Reply on Mailbox

### Check Current Auto Reply

```
Get-MailboxAutoReplyConfiguration user@domain.com
```

<br>

### Set Auto Reply

```
Set-MailboxAutoReplyConfiguration user@domain.com `
  -AutoReplyState Enabled `
  -InternalMessage "This mailbox is no longer monitored." `
  -ExternalMessage "This mailbox is no longer monitored."
```