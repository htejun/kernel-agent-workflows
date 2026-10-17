# Mail sending setup

- **Load when:** the user has no setup for sending mail from their account,
  asks how to set one up, or [mailbox/mail-send](../mailbox/mail-send.md)
  needs a tool that is missing.
- **Parameters:** `$SENDMAIL`, `$MAIL_ACCOUNTS`, `$SELF_COPY`, `$MAIL_AUDIT`,
  `$SENT_LOG`
- **Depends on:** [mail/setup](../mail/setup.md),
  [mailbox/mail-send](../mailbox/mail-send.md)

A reference setup for users who have none: git send-email and msmtp send,
and the profile's mail parameters describe the result. A user with a
working setup keeps it and records it in the profile and overlays.

## Procedure
1. Set up git send-email as [mail/setup](../mail/setup.md#procedure)
   describes, and install msmtp from the distribution's packages.
2. Configure one msmtp account per From address in `~/.msmtprc`, with a
   log file, and the account named `default`
   ([msmtp](#msmtp-configuration)).
3. Point git send-email at msmtp
   ([git send-email](#git-send-email-configuration)).
4. Send one test mail to yourself through each account. Check that it
   arrives outside spam and that the log has a line for it.
5. Fill in the profile: `$SENDMAIL` is `msmtp`, `$MAIL_ACCOUNTS` maps each
   From address to its msmtp account, `$SELF_COPY` is the address that
   keeps a copy of everything sent, `$MAIL_AUDIT` stays at its default, and
   `$SENT_LOG` names a file for the
   [sent log](../mailbox/sent-log.md).

## Reference
### msmtp configuration
```
defaults
auth on
tls on
logfile ~/.msmtp.log

account work
host smtp.example.org
port 587
from jane@example.org
user jane
passwordeval <command that prints the password, such as a keyring lookup>

account default : work
```

- `from` is the address the account sends as. With `-f <address>`, msmtp
  picks the first account whose `from` matches.
- `logfile` makes msmtp append one line per mail, including its
  recipients. [core/publishing](../core/publishing.md#before-publishing)
  checks this log before every send.
- msmtp removes a `Bcc:` header from the message it sends and still
  delivers to the address, which is how a mail file carries the self-copy.
- With `--read-recipients`, msmtp sends to the To, Cc and Bcc headers of
  the file.

### git send-email configuration
git send-email itself and its From identity are set up by
[mail/setup](../mail/setup.md#procedure). For sending, point it at msmtp:

```
git config --global sendemail.sendmailCmd msmtp
git config --global sendemail.envelopeSender auto
```

- `sendmailCmd` makes git send-email hand each message to msmtp instead of
  speaking SMTP itself.
- `envelopeSender auto` passes the From address to msmtp as the envelope
  sender, so msmtp picks the account that matches it.
