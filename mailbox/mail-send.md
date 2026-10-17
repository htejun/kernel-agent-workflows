# Sending mail

- **Load when:** sending mail from the user's account: a patch, a patchset,
  a reply, an acknowledgment or a pull request.
- **Parameters:** `$SEND_TOKEN`, `$SENDMAIL`, `$MAIL_ACCOUNTS`, `$SELF_COPY`,
  `$MAIL_AUDIT`
- **Depends on:** [core/publishing](../core/publishing.md),
  [mail/mail-compose](../mail/mail-compose.md),
  [mailbox/sent-log](../mailbox/sent-log.md)

## Always-on rules
- [GATE] Before sending mail: the publishing gate of
  [core/publishing](../core/publishing.md#before-publishing) passed, the
  files and the command follow
  [mail/mail-compose](../mail/mail-compose.md#before-mail-files-leave-the-agent),
  the account and the self-copy are right, and $MAIL_AUDIT heads the send
  command with `&&`, because a mail sent wrong can't be recalled.
- [GATE] Before reporting a mail as sent: the post-send audit found every
  intended recipient among the envelope recipients, because a send that
  exits 0 can still have dropped recipients.

## [GATE] Checklist
### Before sending mail
- [ ] The publishing gate passed
      ([core/publishing](../core/publishing.md#before-publishing)).
- [ ] The files and the command passed the composing gate
      ([mail/mail-compose](../mail/mail-compose.md#before-mail-files-leave-the-agent)).
- [ ] The account matches each file's From, and the self-copy is in the
      file's Bcc header for $SENDMAIL or in `--bcc` for git send-email
      ([Accounts and the self-copy](#accounts-and-the-self-copy)).
- [ ] The command runs $MAIL_AUDIT on every file and chains the send with
      `&&` ([Audits](#audits)).

### Before reporting a mail as sent
- [ ] The post-send audit passed ([Audits](#audits)). Exit status 0 with a
      missing recipient is a failed send.

## Procedure
### The send flow
1. Build the files and the command by
   [mail/mail-compose](../mail/mail-compose.md). For a pull request, the
   request-pull output goes in the file first
   ([maintain/pull-request](../maintain/pull-request.md#creating-and-sending)).
2. Add the account and the self-copy, and run $MAIL_AUDIT on every file.
   Fix every finding before showing anything.
3. Review the body and the recipients. Present the readiness summary with
   the [mail details](#the-readiness-summary-for-mail) and wait, as
   [core/publishing](../core/publishing.md#procedure) describes.
4. After $SEND_TOKEN, run the presented command: $MAIL_AUDIT, then `&&`
   and the send.
5. Run the post-send audit. Only then report the mail as sent.
6. Record a posted patch, patchset or pull request per
   [mailbox/sent-log](../mailbox/sent-log.md).

### Sending patches with git send-email
Present one shell call that audits and sends the command
[mail/mail-compose](../mail/mail-compose.md#the-git-send-email-command)
built, with `--bcc=$SELF_COPY` added when $SELF_COPY is set:

```
$MAIL_AUDIT --format-patch --from-account <address>=<account> --account <account> \
    --subject '<exact subject of the first file>' /path/to/mails/0000-cover-letter.patch \
    /path/to/mails/0001-first.patch /path/to/mails/0002-second.patch \
&& git send-email --confirm=never --to=<address> --cc=<address> --cc=<list> \
    --bcc=$SELF_COPY /path/to/mails/0000-cover-letter.patch \
    /path/to/mails/0001-first.patch /path/to/mails/0002-second.patch
```

### Sending other mail with the sendmail tool
1. Send every mail that is not format-patch output with $SENDMAIL: replies,
   acknowledgments, discussion, forwards, pings and pull requests.
2. Pick the account from $MAIL_ACCOUNTS by the file's From. Put the
   self-copy in a `Bcc:` header, which msmtp delivers to and removes from
   the message.
3. Present one shell call that audits and sends. With msmtp, the default
   $SENDMAIL:

   ```
   $MAIL_AUDIT --from-account <address>=<account> --from-account <address>=<account> \
       --account <account> --self-copy $SELF_COPY --subject '<exact subject>' \
       [--reply-to /path/to/original.eml] /path/to/mail.txt \
   && $SENDMAIL -a <account> --read-recipients < /path/to/mail.txt
   ```

   A reply or an acknowledgment passes `--reply-to` with the original
   message file, so that the audit checks In-Reply-To against the
   original's Message-ID and finds every recipient of the original kept.
   In both audit commands, pass one `--from-account` for every entry of
   $MAIL_ACCOUNTS, so that a From that belongs to another account fails.
   Leave out `--self-copy` when $SELF_COPY is unset. Another sendmail tool
   takes its own options for the account and for reading recipients from
   headers.

## Reference
### The readiness summary for mail
Besides what
[core/publishing](../core/publishing.md#the-readiness-summary) requires, it
states:
- the recipients: To, Cc and Bcc;
- the exact Subject;
- the threading: enabled for a patchset, what any In-Reply-To points at,
  and the thread-root question when
  [mail/thread-roots](../mail/thread-roots.md#procedure) leaves the choice
  to the user;
- the send tool and the account, matching the file's From;
- the first line of each file, quoted.

### Accounts and the self-copy
- The sendmail account matches the file's From, per $MAIL_ACCOUNTS. Mail
  with a From in one domain sent through another domain's server fails
  DMARC alignment: recipients' providers file it as spam, and no copy
  reaches the From address's sent folder.
- Every outgoing mail copies $SELF_COPY when it is set, patches and other
  mail alike. A send that omits it is incomplete even when it reached the
  lists. The self-copy is the sender's copy, so the sending address stays
  out of To and Cc.

### Audits
- $MAIL_AUDIT checks the files before they are presented and again at the
  head of the send command, where `&&` makes a finding stop the send. Its
  `--help` lists the checks. Its output shows each file's first line,
  Subject, path and modification time next to the result of the send.
- The post-send audit compares recipients with a mail library, never by
  eye. git send-email prints the envelope recipients of each message: the
  sendmail command line, or RCPT TO lines when it talks SMTP itself. Parse
  the To, Cc and Bcc headers of the files, add the `--to`, `--cc` and
  `--bcc` addresses of the command, and check that every one of them is
  among the envelope recipients. msmtp prints nothing on success, so for a
  $SENDMAIL send, the recipients $MAIL_AUDIT printed and the exit status 0
  of the single run are the post-send audit.

### msmtp has no dry run
`msmtp -d` is verbose output, not a dry run, and msmtp has no dry-run
option.

## Pitfalls
- Mail with a From at one provider went out through another provider's
  server, and every recipient at the From's provider got it as spam.
- An acknowledgment went out three times because the send command was
  re-run to inspect it.

## Customization
- `$MAIL_ACCOUNTS` and `$SELF_COPY` describe the accounts.
  [mailbox/send-setup](../mailbox/send-setup.md) describes a reference
  setup.
- An overlay records where the sendmail log is and the flags the user's
  $MAIL_AUDIT takes when it isn't this repository's tool.
