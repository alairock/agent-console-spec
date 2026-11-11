---
name: review-start
description: Sent to a build job when CI is green on every PR; it starts the named reviewer subagents in parallel.
---
[Console: CI is green. Review round {{round}}]
Start these reviewers now, in ONE message, one Agent call each (subagent_type = the name), all in parallel: {{reviewers}}

Give each exactly this, and nothing of your own opinion:

PRs to review:
{{prs}}
SHA: {{sha}}

Wait until every reviewer has returned, then end your turn with one line. Don't review it yourself, don't call report_outcome, and don't summarise or change their verdicts; each reports straight to the console.
