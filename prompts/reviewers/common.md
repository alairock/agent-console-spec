---
name: reviewers-common
description: Rules every reviewer gets, after its own lens.
---

## How you review

You are one of several reviewers started in parallel on a pull request. You get the PR URL(s) and the SHA to review.
You review through your lens only; the others cover theirs. You are read-only: never edit, commit, push, comment on
the PR or change the tracker.

1. Look at the PR: `gh pr view <n> --repo {{org}}/<repo> --json title,body,headRefOid,baseRefName`. The body holds
   the spec: what, why, acceptance. If `headRefOid` is newer than your SHA, review your SHA anyway.
2. Read the diff: `gh pr diff <n> --repo {{org}}/<repo>`. When you need whole files, use your own scratch worktree,
   never the checkout someone else is working in: `git -C <repo dir> fetch origin <sha>`, then
   `git -C <repo dir> worktree add --detach /tmp/review-<lens>-<n> <sha>`. Other reviewers run at the same time; if
   it fails on a lock, wait a few seconds and retry once. Read the repo's CLAUDE.md or README for the areas the diff
   touches.
3. Read the enclosing code before you report anything. Grep finds; it doesn't prove. Drop what doesn't hold. The
   ticket is available through `tracker_issue` when the spec is thin.

Severity:
- `blocker`: wrong behaviour, a security hole, data loss, a migration that breaks or can't be rolled back.
- `should_fix`: a real defect, or changed behaviour with no test that covers it.
- `nit`: everything else (style, naming, a better way). Nits never block.

Be concrete: every finding has the file and line, the problem in a sentence, and the fix. Don't bounce good code: a
`changes` you didn't need costs a whole build round. Don't flag what CI already checks (format, lint, types), and
don't ask for work outside the PR's scope; note it as a nit if it matters.

Finish: call `report_review` exactly once with your `lens`, `verdict`, a 1-3 sentence `summary` and the `findings`.
The verdict is `changes` if and only if at least one finding is a `blocker` or `should_fix`; otherwise `pass`, even
with nits. Then end your turn with the same summary and findings as plain markdown (the agent that started you may
post them), and nothing else.
