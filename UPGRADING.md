# Upgrading

Changes that need action in a user's profile or overlays, newest first. Each
entry has a short heading naming the change, such as "New module
review/checkpatch", and says what to do. To see the entries added since your
last update, follow [Updating](INTEGRATION.md#updating).

## New parameter $MODEL_CHECKER, tla-mctj by default

review/formal-model-check now runs model checks on tla-mctj by default and
on TLC for what tla-mctj can't do. Install tla-mctj as
[Installation](review/formal-model-check.md#installation) describes, or set
`$MODEL_CHECKER` to `tlc` in the profile to keep TLC throughout.

## Initial version

Nothing to do. Follow INTEGRATION.md to set up.
