# Mail setup

- **Load when:** the user has no setup for reading lore threads or sending
  mail, asks how to set one up, or a mail module needs a tool that is
  missing.
- **Parameters:** `$MAIL_READ`, `$SENDMAIL`, `$MAIL_ACCOUNTS`, `$SELF_COPY`,
  `$MAIL_AUDIT`, `$SENT_LOG`, `$IDENTITY`
- **Depends on:** [mail/mail-send](../mail/mail-send.md)

A reference setup for users who have none: b4 and lei read from lore, git
send-email and msmtp send, and the profile's mail parameters describe the
result. A user with a working setup keeps it and records it in the profile
and overlays.

## Procedure
1. Install b4, git send-email and msmtp from the distribution's packages.
   lei comes with public-inbox and is optional. Some distributions ship git
   send-email in a separate package.
2. Configure one msmtp account per From address in `~/.msmtprc`, with a
   log file, and the account named `default`
   ([msmtp](#msmtp-configuration)).
3. Point git send-email at msmtp and set the identity
   ([git send-email](#git-send-email-configuration)).
4. Send one test mail to yourself through each account. Check that it
   arrives outside spam and that the log has a line for it.
5. Fill in the profile: `$MAIL_READ` is `lore`, `$SENDMAIL` is `msmtp`,
   `$MAIL_ACCOUNTS` maps each From address to its msmtp account,
   `$SELF_COPY` is the address that keeps a copy of everything sent,
   `$MAIL_AUDIT` stays at its default, and `$SENT_LOG` names a file for the
   [sent log](../mail/sent-log.md).

## Reference
### Reading with b4 and lore
- `b4 am <msgid>` writes an mbox of the series, ready for `git am`, with
  the trailers from the replies added. `b4 mbox <msgid>` writes the whole
  thread. [mail/lore-thread](../mail/lore-thread.md#procedure) shows how
  the two are used and the lore URLs.
- lei keeps searches of lore in local mailboxes. `lei q` runs a query: `-I
  https://lore.kernel.org/all/` searches lore, `-o <dir>` writes the
  results, and `-t` adds every message of each matching thread. `lei up
  <dir>` updates a saved search. lei-q(1) describes the query syntax.

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
  recipients.
  [mail/mail-send](../mail/mail-send.md#running-a-send-once) treats this
  log as the record of what went out.
- msmtp removes a `Bcc:` header from the message it sends and still
  delivers to the address, which is how a mail file carries the self-copy.
- With `--read-recipients`, msmtp sends to the To, Cc and Bcc headers of
  the file.

### git send-email configuration
```
git config --global sendemail.sendmailCmd msmtp
git config --global sendemail.from "$IDENTITY"
git config --global sendemail.envelopeSender auto
```

- `sendmailCmd` makes git send-email hand each message to msmtp instead of
  speaking SMTP itself.
- `envelopeSender auto` passes the From address to msmtp as the envelope
  sender, so msmtp picks the account that matches it.
- `git send-email --dry-run` shows each message and its recipients without
  sending.

## Customization
- A user with a local mailbox, such as a synced maildir with a search
  index, sets `$MAIL_READ` to a name for it and describes it in the overlay
  of [mail/lore-thread](../mail/lore-thread.md): how to search, how to read
  a message, and its quirks.
