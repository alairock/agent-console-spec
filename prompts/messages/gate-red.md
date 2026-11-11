---
name: gate-red
description: Sent to a build job when CI fails on the PR head it reported. {{failures}} has one section per failed check with its log tail.
---
[Console: CI failed, gate round {{round}} of {{max}}]
{{failures}}

Fix what these logs show, run the tests you touched by name, push, and call report_outcome to_review again with the same pr_urls, risk and reviewers. Don't watch CI. After round {{max}} the ticket goes to a person.
