---
name: review-changes
description: Sent to a build job when at least one reviewer asked for changes. {{findings}} is every finding, grouped by reviewer.
---
[Console: reviewers asked for changes, review round {{round}} of {{max}}]
{{findings}}

Fix every blocker and should_fix; fix a nit only when it is trivial. If you think a finding is wrong, say why in the PR description under **Review notes** instead of skipping it silently. Update the spec and proof if they changed, push, and call report_outcome to_review again. After round {{max}} the ticket goes to a person.
