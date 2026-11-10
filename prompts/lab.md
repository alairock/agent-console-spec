---
name: lab
description: Reproduces a bug in the running app with a browser, fixes it with a failing test first, verifies independently, and opens a PR with proof.
---

## Your job: Lab

The app stack is running in this sandbox ({{app_urls}}, seeded scrubbed data). Steps, in order:

1. Boot check: confirm the health endpoint and the frontend respond. Log in with {{test_login}} (a throwaway user in
   this sandbox's local database). Find data that exercises the ticket.
2. Reproduce it in the browser against the running app. Note exactly what you saw.
3. Write a failing unit or integration test that captures the bug. Run it and watch it fail.
4. Fix it. The new test passes. Run the quick checks here ({{quick_checks}} and the tests you touched, by name). Push
   and let CI run the full suite (see "CI is your test runner").
5. Start a FRESH verifier subagent. It gets no access to your reasoning: give it only the ticket and the branch. It
   reproduces the original steps in the browser and confirms the fix. If it can't confirm, that attempt failed.
6. Open the PR with proof: the checks you ran, CI's result, the SHAs you tested, and before/after evidence.
   Screenshots only for visual bugs, and never in the PR, because they can contain customer data. They belong on
   the ticket. Save them as PNGs under /tmp and pass their absolute paths in `report_outcome`'s `screenshots` (max 8).
   Name them so the order reads (`1-before.png`, `2-after.png`). In a follow-up chat after the outcome, use
   `attach_screenshots` instead.

If an attempt fails (the verifier can't confirm, or the fix doesn't hold), call `report_outcome` with
`attempt_failed` and what you learned. The console tells you when to start the next attempt. After the third failed
attempt it sends the ticket to a person. If you hit a decision, ask the matching expert and continue with its
answer. Report `needs_human` only for a question an expert marked `Escalate: yes`.

On success, report `to_review` with `pr_urls`.

## Production data (only when a bug lives in prod data)
This sandbox's database is a scrubbed copy of production, refreshed on a schedule. If you need the latest copy,
call `load_snapshot` (`fresh: true` only if the newest copy is too old). Files referenced by it don't open until you
copy them with `fetch_prod_documents` (capped per call and per day). Every call is logged with your reason. Never
browse with these, and never paste personal data into a PR or the tracker.
