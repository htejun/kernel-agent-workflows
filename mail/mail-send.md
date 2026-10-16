# Sending mail and publishing

- **Load when:** sending any mail, or publishing anything public, such as a
  push to a shared remote or a GitHub post.
- **Parameters:** `$SEND_TOKEN`, `$SENDMAIL`, `$MAIL_ACCOUNTS`, `$SELF_COPY`,
  `$MAIL_AUDIT`
- **Depends on:** [core/safe-operations](../core/safe-operations.md),
  [mail/thread-roots](../mail/thread-roots.md),
  [mail/sent-log](../mail/sent-log.md)

## Always-on rules
- [GATE] Before a public send: the files and the command pass their checks,
  a readiness summary presented the exact command, and the user's most
  recent message contains $SEND_TOKEN as a standalone word, because mail,
  GitHub posts and pushes to a shared remote, a repository other people
  work in, can't be recalled. The push of an instructed apply and the
  signed tag before a pull request don't wait for $SEND_TOKEN.
- "Post", "send", "ship", "go ahead", "do it", "yes" and "ok" mean prepare
  and wait, because only $SEND_TOKEN approves a send.
- Never run a send command expecting a permission prompt to stop it,
  because running the command is the send and the harness may allow the
  command without asking.
- Finish every step but the send before asking for $SEND_TOKEN, because
  anything composed, pushed or assembled after the approval was never
  approved.
- [GATE] Before reporting a mail as sent: the post-send audit found every
  intended recipient among the envelope recipients, because a send that
  exits 0 can still have dropped recipients.
- Never edit a mail file once it is sent. A follow-up goes in a new file,
  and a file whose send status is unknown counts as sent, because the file
  is the record of what went out.

## [GATE] Checklist
### Before a public send
- [ ] Every step except the send command is done: the files are complete,
      tags are signed and pushed, request-pull output is in the file.
- [ ] Every path in the command is absolute, and every file is named
      ([Mail files](#mail-files)).
- [ ] Format-patch output goes through git send-email with the recipients
      on the command line, `--confirm=never`, the self-copy as `--bcc` and
      no `--suppress-cc`, and its dry run listed the full recipient set on
      every message ([git send-email](#sending-patches-with-git-send-email)).
- [ ] A patchset goes out threaded, in one invocation.
- [ ] Other mail goes through $SENDMAIL with the account matching its From
      and the self-copy in its Bcc header
      ([the sendmail tool](#sending-other-mail-with-the-sendmail-tool)).
- [ ] The thread root follows [mail/thread-roots](../mail/thread-roots.md).
- [ ] The readiness summary was presented with the complete files
      ([The readiness summary](#the-readiness-summary)).
- [ ] The user's most recent message, re-read just now, contains
      $SEND_TOKEN as a standalone word, or the
      [second-factor retry](#second-factor-tokens) applies.
- [ ] This file set was not sent before. The sendmail log is the record of
      what went out.
- [ ] The command runs $MAIL_AUDIT on every file and chains the send with
      `&&` ([Audits](#audits)).
- [ ] The command is the presented string, character for character. Any
      difference means stop, present the actual command, and wait for a
      fresh $SEND_TOKEN.

### Before reporting a mail as sent
- [ ] The post-send audit passed ([Audits](#audits)). Exit status 0 with a
      missing recipient is a failed send.

## Procedure
### The send flow
1. Prepare everything. Compose each mail as a complete file
   ([Mail files](#mail-files)) and choose its thread root by
   [mail/thread-roots](../mail/thread-roots.md). For a pull request, sign and
   push the tag and put the request-pull output in the file first
   ([maintain/pull-request](../maintain/pull-request.md#creating-and-sending)).
2. Run $MAIL_AUDIT on every file, and the dry run on format-patch output.
   Fix every finding before showing anything.
3. Review the body and the recipients. Present the readiness summary
   alongside the complete files.
4. Wait. Before running the send, re-read the user's most recent message
   for $SEND_TOKEN.
5. Run the presented command: $MAIL_AUDIT, then `&&` and the send.
6. Run the post-send audit. Only then report the mail as sent.
7. Record a posted patch, patchset or pull request per
   [mail/sent-log](../mail/sent-log.md).

### Sending patches with git send-email
1. Use git send-email for format-patch output only: a single patch or a
   patchset. Never send a hand-written reply file with it.
2. git send-email takes recipients per message. A cover letter's To and Cc
   don't reach the patches. Put every recipient on the command line with
   `--to` and `--cc`. Who the recipients are is in
   [submit/patchset-posting](../submit/patchset-posting.md#recipients).
3. Add `--confirm=never`, because git send-email adds Cc addresses from tag
   lines and then stops at a confirmation prompt that a non-interactive run
   can't answer.
4. Add `--bcc=$SELF_COPY` when $SELF_COPY is set. git send-email keeps a
   Bcc header in a patch file as a visible header and doesn't deliver to
   it, so the files carry none.
5. Never pass any form of `--suppress-cc`, because it can drop addresses the
   files name.
6. Send a patchset in one invocation: the cover letter and every patch,
   threaded. Never `--no-thread`, never a loop over the patches, never the
   cover and the patches as separate commands, and never $SENDMAIL on
   format-patch output, because only one invocation keeps the thread and
   the recipients consistent.
7. Run the git send-email command with `--dry-run` and stdin from
   /dev/null, and check that every message lists the full recipient set.
8. Present one shell call that audits and sends:

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

   In both audit commands, pass one `--from-account` for every entry of
   $MAIL_ACCOUNTS, so that a From that belongs to another account fails.
   Leave out `--self-copy` when $SELF_COPY is unset. Another sendmail tool
   takes its own options for the account and for reading recipients from
   headers.

## Reference
### What counts as publishing
- Mail through git send-email or $SENDMAIL.
- GitHub: creating a pull request or an issue, commenting on either,
  reviewing or approving a pull request, a public gist, a release, an
  attachment upload, a POST to a public API endpoint.
- Publishing a package, such as `cargo publish`, and web hooks.
- A push to a shared remote: a release tag, a new branch, an update or a
  force-push. The pushes that don't wait belong to
  [maintain/apply-patches](../maintain/apply-patches.md#applying) and
  [maintain/pull-request](../maintain/pull-request.md#creating-and-sending).
- Anything else publicly visible or indexable. A draft shown to the user
  publishes nothing.

### The readiness summary
Present it alongside the complete files. It states:
- the recipients: To, Cc and Bcc;
- the exact Subject;
- the threading: enabled for a patchset, what any In-Reply-To points at,
  and the thread-root question when
  [mail/thread-roots](../mail/thread-roots.md#procedure) leaves the choice
  to the user;
- the send tool and the account, matching the file's From;
- the first line of each file, quoted;
- the exact shell command, $MAIL_AUDIT included: absolute paths, every
  option, every file named, no `...` and no globs. For a force-push, the
  old and the new tip of the branch.

$SEND_TOKEN approves that command string, not the prose around it.

### Second-factor tokens
- Some services, such as a git server that wants a second factor for
  pushes, reject a command until the user supplies a token. Try the
  command first: a token given earlier often stays valid for a while and
  covers several pushes.
- On an authentication error, ask the user for the token as plain text.
  Validate it in a separate, non-interactive command. Never wait at an
  interactive prompt.
- Then retry the unchanged command. When the command was approved with
  $SEND_TOKEN, the token keeps that approval, so don't ask for $SEND_TOKEN
  again. The audits still apply, and a send that completed is never
  repeated.

### Mail files
- Compose the complete mail file directly and check it with $MAIL_AUDIT.
  Don't write per-message generators, audit scripts, body intermediates or
  manifests, because each is new code nobody reviewed. When a check is
  missing, extend the shared tool.
- The first line of mail for $SENDMAIL is a header. Format-patch output
  starts with an mbox separator line,
  `From <commit-id> Mon Sep 17 00:00:00 2001`. git send-email strips it,
  but msmtp sends it as is, and the message goes out with no Subject, To or
  Cc.
- Every path in a send command is absolute and every file is named, or
  listed in an explicit brace list. Never a glob, and never `cd <dir> &&`
  followed by bare file names, because a stale file in the current
  directory or one that matches the glob silently replaces the intended
  one.
- To, Cc and Bcc each appear once, with comma-separated addresses. Of the
  line limits, only RFC 5322's 998 columns per header line is enforced.
- A non-ASCII display name is RFC 2047 encoded, never transliterated or
  dropped: `=?UTF-8?Q?Ren=C3=A9_Example?= <rene@example.org>`.
- The signature separator is `-- ` with a trailing space. Editors and
  file-writing tools can strip the space, and $MAIL_AUDIT rejects the bare
  `--` that is left.
- A follow-up to a sent file goes in a new numbered file: when `reply2`
  is sent, write `reply3`.

### Recipients
- A reply never drops a recipient of the message it answers from To or
  Cc. Moving an address to Bcc drops it. Adding is fine.
- In mail for $SENDMAIL, the sending address never goes in To or Cc, even
  when the original message listed it there. The self-copy is the sender's
  copy. git send-email copies the From and Signed-off-by addresses of a
  patch by default.

### Accounts and the self-copy
- The sendmail account matches the file's From, per $MAIL_ACCOUNTS. Mail
  with a From in one domain sent through another domain's server fails
  DMARC alignment: recipients' providers file it as spam, and no copy
  reaches the From address's sent folder.
- Every outgoing mail copies $SELF_COPY when it is set, patches and other
  mail alike. A send that omits it is incomplete even when it reached the
  lists.

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

### Running a send once
Every run of a send command sends again
([core/safe-operations](../core/safe-operations.md#always-on-rules)).
`msmtp -d` is verbose output, not a dry run, and msmtp has no dry-run
option. The sendmail log shows what went out.

## Pitfalls
- An acknowledgment went out with no approval: the agent ran the send
  expecting a permission prompt to stop it, and the prompt allowed it.
- Pull request branches were force-pushed after a CI failure report,
  without asking.
- `--suppress-cc=all` once stripped a file's Cc header. Several recipients
  were dropped, and a duplicate had to go out to reach them.
- Mail with a From at one provider went out through another provider's
  server, and every recipient at the From's provider got it as spam.
- A format-patch file piped into msmtp went out with no Subject, and the
  list rejected it.
- A cover letter written days earlier sat in the current directory and
  went out instead of the new one.
- An acknowledgment went out three times because the send command was
  re-run to inspect it.
- Two replies went out with a bare `--` signature separator.

## Customization
- `$SEND_TOKEN` sets the approval word. `$MAIL_ACCOUNTS` and `$SELF_COPY`
  describe the accounts. [mail/setup](../mail/setup.md) describes a
  reference setup.
- An overlay records which remotes are shared, where the sendmail log is,
  how the user's services validate a second-factor token, and the flags
  the user's $MAIL_AUDIT takes when it isn't this repository's tool.
