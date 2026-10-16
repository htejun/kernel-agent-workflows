# Sent log

- **Load when:** a patch, a patchset or a pull request was just posted, or a
  task needs the Message-ID, base or version history of an earlier posting.
- **Parameters:** `$SENT_LOG`
- **Depends on:** [mail/mail-send](../mail/mail-send.md)

$SENT_LOG is the record of what the user posted: the place to find a
previous version's Message-ID and base before posting the next one. The
module applies when $SENT_LOG is set.

## Always-on rules
- Take the Message-ID of a posting from the send itself or the local sent
  copy, never from lore, because lore can lag the send and a guessed ID is
  wrong.

## Procedure
1. After the post-send audit of
   [mail/mail-send](../mail/mail-send.md#audits) passes, record each posted
   patch, patchset and pull request in $SENT_LOG.
2. Take the Message-ID from the git send-email output. For a mail sent with
   the sendmail tool, take it from the local sent copy.
3. Add an entry for a new patch or patchset, or add the new version under
   the existing entry when one exists. Never start a second entry for the
   same patchset.
4. Record a pull request with the fields below, under its own entry.

## Reference
### Entry formats
A patch or patchset:

```
### "<title>"
- **v<N>** (<date>)
  - Message-ID: <msgid>
  - Link: https://lore.kernel.org/r/<msgid>
  - Base: <tree>/<branch> (<sha12>)
  - Branch: <branch the patchset was posted from>
  - Patches: <count>
  - Git tree: <URL and branch, when published>
```

A pull request:

```
### <subject of the pull request>
- **Tag**: <signed tag>
- **Tree**: <URL of the tree>
- **Branch**: <branch>
- **Date**: <date>
- **Patches**: <count>
```

## Customization
- `$SENT_LOG` is the file. An overlay can split patchsets and pull requests
  into separate files or add fields.
