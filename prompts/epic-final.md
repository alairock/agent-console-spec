---
name: epic-final
description: Added to the build job (started with the app up) that is an epic's final whole-feature check. {{base_branch}} is the epic's feature branch.
---

## This is an epic's final job
All of the epic's sub-issue PRs have merged into `{{base_branch}}`. Check out `{{base_branch}}` in every repo that has
it, check the whole feature in the running app against the parent's plan, fix what doesn't hold, and open ONE PR from
`{{base_branch}}` to main. Its spec is the parent's plan. Report it like any build (risk, reviewers): it goes through
the gate, the reviewers and the approver as usual.
