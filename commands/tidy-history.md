---
description: Reset the current branch to a base ref and rebuild its history into clean, logically-grouped commits
argument-hint: [base-ref (default: main)] [--dry-run]
allowed-tools: Bash, Read, Edit, Write, Grep, Glob
---

You are tidying the commit history of the current git branch. Parse `$ARGUMENTS`: a token equal to `--dry-run` enables dry-run mode; any other token is the base ref (default to `main` if none given).

Follow this procedure exactly and report at each gate:

1. **Review.** Run `git branch --show-current`. If it is `main`, `master`, or equal to the resolved base ref, STOP — this command must run on a feature branch, not on the trunk or the base itself. Run `git status` (must be clean — if not, STOP and tell the user to commit/stash first). Compute `MERGE_BASE=$(git merge-base <base> HEAD)` once, and use `$MERGE_BASE` (not `<base>` directly) for every comparison and reset below — this scopes the tidy to only what the branch actually added, even if `<base>` has since moved forward. Review the full delta the branch adds: `git log $MERGE_BASE..HEAD --oneline`, `git diff $MERGE_BASE...HEAD --stat`, and read the actual diff. Summarize the logical groups of change you see.

2. **Confirm grouping.** Propose a small set of commits — one per logical feature/concern — preserving this repo's existing commit-tag style (inspect recent history with `git log --oneline -20` to detect the convention, e.g. `Added:` / `Changed:` / `Fix:` / `Dev:`). A single file's hunks may belong to different groups; plan to split them with `git add -p` or by staging reconstructed file states. For each group, record the author (`git log --format='%an <%ae>'`) and date (`%aI`) of the most recent original commit contributing to it — this is what the rebuilt commit will carry. Present the plan, the proposed commit messages, and the author/date each will use, and ask the user to approve before touching anything.

   If dry-run mode is enabled, STOP here after presenting the plan. Do not create a backup tag or change anything. Tell the user no changes were made and that dropping `--dry-run` will execute this same plan for real.

3. **Safety net.** Create `git tag backup/<branch>-pre-tidy` at the current HEAD so the original history is recoverable by name.

4. **Reset.** Capture the final file contents first (the working tree already equals HEAD), then `git reset --hard $MERGE_BASE` and rebuild each file's state per group, OR use `git reset --soft $MERGE_BASE` + `git reset` and stage selectively with `git add -p`. Choose whichever cleanly produces partial-file commits.

5. **Re-commit.** Create each commit in dependency order with the approved tag-prefixed messages, passing `--author="<name> <email>"` and `--date="<iso-date>"` from step 2's record so the rebuilt commit keeps the original author and timestamp instead of the person running this command. Keep related changes together; use partial-file staging where a file spans groups.

6. **Verify.** `git diff backup/<branch>-pre-tidy HEAD` MUST be empty (identical end-state). If non-empty, investigate (often trailing-newline/whitespace) and fix until it is empty. Confirm `git diff $MERGE_BASE..HEAD --stat` matches the pre-tidy stat and `git status` is clean. Show the new `git log $MERGE_BASE..HEAD --oneline`.

7. **Stop.** Do NOT push. The branch may have diverged from its remote; updating it requires `git push --force-with-lease`, which you must NOT run without the user explicitly asking. Report the result and remind the user the original history is at `backup/<branch>-pre-tidy` (recover via `git reset --hard backup/<branch>-pre-tidy`).
