# GitHub pull requests

- **Load when:** opening or updating a GitHub pull request, writing its
  title and body, or commenting on, reviewing or approving one.
- **Parameters:** none
- **Depends on:** [core/git-safety](../core/git-safety.md),
  [review/patchset-review](../review/patchset-review.md),
  [mail/mail-replies](../mail/mail-replies.md),
  [mail/mail-send](../mail/mail-send.md)

## Always-on rules
- [GATE] Before opening a GitHub pull request: the branch builds, the
  repository's formatter and lock files are current, the patchset review
  ran unless the user said "skip review", and the user saw the title and
  body, because CI and reviewers act on the first push.
- A pull request body is a short summary and an itemized commit list,
  because a dense body on a web page gets skipped.

## [GATE] Checklist
### Before opening a GitHub pull request
- [ ] The branch builds, and the repository's formatter and lock files are
      current.
- [ ] The patchset review ran, as
      [review/patchset-review](../review/patchset-review.md#running-a-milestone-review)
      describes, unless the user said "skip review".
- [ ] The title and body were shown to the user together with the push and
      create commands, under the send gate in
      [mail/mail-send](../mail/mail-send.md#before-a-public-send).

## Procedure
### Opening or updating a pull request
1. Put the commits on a pull request branch based on the upstream branch.
   Development and backup branches never go to the upstream repository.
2. Build, and bring the formatter's output and the lock files up to date.
   Fold a fix to an earlier commit of the branch, such as a lock file
   update, into that commit. The fixup commit needs the commit gate in
   [core/git-safety](../core/git-safety.md#before-a-commit-or-amend).
   `git rebase --autosquash` runs without `-i` from git 2.44 on:

   ```
   git commit --fixup=<commit> <files>
   git rebase --autosquash <base>
   ```

3. Write the title and the [body](#pull-request-bodies).
4. Prepare the push. A new branch goes up with `git push <remote> <branch>`,
   to the upstream repository when the user can push there, or to the
   user's fork. An update that rewrites the branch is a force-push
   ([core/git-safety](../core/git-safety.md#branch-structural-operations)) and
   uses `--force-with-lease=<branch>:<old tip>`, so that the push fails if
   anyone else pushed in between.
5. Prepare `gh pr create` with the title and body. From a fork, name the
   upstream repository and the fork's branch:
   `gh pr create --repo <upstream> --head <owner>:<branch>`.
6. Present the push and `gh pr create` commands with the title and body, under
   the send gate in [mail/mail-send](../mail/mail-send.md#before-a-public-send).
   The gate covers every post here: the push, the pull request, comments,
   reviews and approvals.

### Reviewing and approving
1. Approve with `gh pr review <number> --approve` and no body. Add text
   only for a substantive point.
2. Write review comments as [review comments](#review-comments) describes.

## Reference
### Pull request bodies
A pull request body is read on a web page, not in a mail client.
- Short paragraphs. The commits or changes go in an itemized list, one item
  each, never a paragraph that walks through them. No "Test plan"
  checkboxes.
- High level and in the imperative, like a commit subject expanded: "Add a
  NUMA-aware mode to the scheduler." Say what the change gives the user and
  what it depends on, never the mechanism, such as which lock or which path.
  No follow-up or future work.
- No Testing section unless the user asks for it or the change needs a
  testing claim to be reviewable, such as a campaign result the pull
  request exists to report. Build and format checks, "ran in a fork" and
  "not run here" never justify it.
- When testing belongs, it is a plain compact paragraph after the summary,
  two or three sentences: the kinds of coverage in one clause, and the
  change's own open issues with one clause each. Never phase names, option
  combinations, machine thread counts, minute counts or per-issue numbers.
  A number stays only when it is the claim and public, such as a verifier
  limit or a budget fraction. Build, format and tree-identity checks go
  without saying. A failure that reproduces without the change is
  unrelated and goes unmentioned.

### Review comments
Tone and length follow
[mail/mail-replies](../mail/mail-replies.md#replies). On a pull request:
- Match the scope the user asked for. When asked only to note problems,
  report observations without adding requests, fixes or CI requirements.
- No list of what is good, no cost and benefit case for a nit, no answers
  to counter-arguments nobody made, and no long style rationale.
- A review the user wants marked as AI-generated takes the form in
  [mail/mail-replies](../mail/mail-replies.md#drafts-under-the-users-name).

## Pitfalls
- A first draft of a body narrated the mechanism and announced follow-up
  work, and took three rounds to reach a high-level summary in the
  imperative. Start there.

## Customization
- An overlay usually names the upstream and backup remotes of each
  repository, the formatter and lock-file commands, any local hook a
  force-push must satisfy, and any repository guide line this module
  overrides.
