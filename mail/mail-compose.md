# Composing mail

- **Load when:** building the files and the send command for any mail, such
  as a patch, a patchset, a reply, an acknowledgment or a pull request,
  whether the agent or the user sends it.
- **Parameters:** none
- **Depends on:** [mail/thread-roots](../mail/thread-roots.md)

Who sends the result is in
[core/publishing](../core/publishing.md#who-sends-mail).

## Always-on rules
- [GATE] Before mail files leave the agent: the files and the send command
  pass the [composing checklist](#before-mail-files-leave-the-agent),
  whether the agent sends them or the user does, because a mail with a
  wrong header, thread or recipient can't be recalled.
- Never edit a mail file once it is sent. A follow-up goes in a new file,
  and a file whose send status is unknown counts as sent, because the file
  is the record of what went out.

## [GATE] Checklist
### Before mail files leave the agent
- [ ] The first line of every mail for a sendmail tool is a header
      ([Mail files](#mail-files)).
- [ ] Every path in the command is absolute, and every file is named.
- [ ] Format-patch output goes through git send-email with the recipients
      on the command line, `--confirm=never`, no Bcc header in the files
      and no `--suppress-cc`, and its dry run listed the full recipient set
      on every message ([git send-email](#the-git-send-email-command)).
- [ ] A patchset goes out threaded, in one invocation.
- [ ] The thread root follows [mail/thread-roots](../mail/thread-roots.md).
- [ ] A reply keeps every recipient of the message it answers in To or Cc
      ([Recipients](#recipients)).

## Procedure
### Building a mail file
1. Compose the complete mail file directly. Don't write per-message
   generators, audit scripts, body intermediates or manifests, because each
   is new code nobody reviewed. When a check is missing, extend the shared
   check tool, such as this repository's tools/mail-audit.
2. Choose its thread root by [mail/thread-roots](../mail/thread-roots.md)
   and set In-Reply-To and References.
3. Follow the [mail file rules](#mail-files) and the
   [recipient rules](#recipients).
4. Check the signature separator line with `grep -c '^-- $' <file>`. When
   an editor or a file-writing tool stripped its trailing space, restore it
   with an explicit edit of that one line, never an in-place substitution
   over the file
   ([core/safe-operations](../core/safe-operations.md#editing-files)), and
   check again.

### The git send-email command
1. Use git send-email for format-patch output only: a single patch or a
   patchset. Never send a hand-written reply file with it.
2. git send-email takes recipients per message. A cover letter's To and Cc
   don't reach the patches. Put every recipient on the command line with
   `--to` and `--cc`. Who the recipients are is in
   [submit/patchset-posting](../submit/patchset-posting.md#recipients).
3. Add `--confirm=never`, because git send-email adds Cc addresses from tag
   lines and then stops at a confirmation prompt that a non-interactive run
   can't answer.
4. A blind copy goes on the command line as `--bcc`. git send-email keeps a
   Bcc header in a patch file as a visible header and doesn't deliver to
   it, so the files carry none.
5. Never pass any form of `--suppress-cc`, because it can drop addresses the
   files name.
6. Send a patchset in one invocation: the cover letter and every patch,
   threaded. Never `--no-thread`, never a loop over the patches, never the
   cover and the patches as separate commands, and never a sendmail tool on
   format-patch output, because only one invocation keeps the thread and
   the recipients consistent.
7. Run the command with `--dry-run` and stdin from /dev/null, and check
   that every message lists the full recipient set. A dry run sends
   nothing:

   ```
   git send-email --dry-run --confirm=never --to=<address> --cc=<address> --cc=<list> \
       /path/to/mails/0000-cover-letter.patch /path/to/mails/0001-first.patch \
       /path/to/mails/0002-second.patch </dev/null 2>&1
   ```

   The send command is the same without `--dry-run`.

## Reference
### Mail files
- The first line of a mail for a sendmail tool is a header: it matches
  `^[A-Za-z][-A-Za-z0-9]*:[ \t]`. Format-patch output starts with an mbox
  separator line, `From <commit-id> Mon Sep 17 00:00:00 2001`. git
  send-email strips it, but msmtp sends it as is, and the message goes out
  with no Subject, To or Cc.
- Every path in a send command is absolute and every file is named, or
  listed in an explicit brace list. Never a glob, and never `cd <dir> &&`
  followed by bare file names, because a stale file in the current
  directory or one that matches the glob silently replaces the intended
  one.
- To, Cc and Bcc each appear once, with comma-separated addresses.
- A header line stays within 998 columns, the hard limit of RFC 5322. The
  78-column limit is only a recommendation, so a longer Subject or body
  line is fine.
- A non-ASCII display name is RFC 2047 encoded, never transliterated or
  dropped: `=?UTF-8?Q?Ren=C3=A9_Example?= <rene@example.org>`.
- The signature separator is `-- ` with a trailing space. Mail readers
  don't treat a bare `--` as a separator.
- A follow-up to a sent file goes in a new numbered file: when `reply2`
  is sent, write `reply3`.

### Recipients
- A reply never drops a recipient of the message it answers from To or
  Cc. Moving an address to Bcc drops it. Adding is fine.
- In mail sent by a sendmail tool, the sending address never goes in To or
  Cc, even when the original message listed it there. The rule doesn't
  cover git send-email, which copies the From and Signed-off-by addresses
  of a patch by default.

## Pitfalls
- `--suppress-cc=all` once stripped a file's Cc header. Several recipients
  were dropped, and a duplicate had to go out to reach them.
- A format-patch file piped into msmtp went out with no Subject, and the
  list rejected it.
- A cover letter written days earlier sat in the current directory and
  went out instead of the new one.
- Two replies went out with a bare `--` signature separator.
