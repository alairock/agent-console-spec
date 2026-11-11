---
name: epic-child
description: Added to any job on a sub-issue of an approved epic. {{base_branch}} is the epic's feature branch.
---

## This ticket is part of an epic
Branch from `{{base_branch}}` and open your PR against it, not main. If `origin/{{base_branch}}` doesn't exist yet in
the repo you change, create it from origin/main and push it first.

Your first message has the parent's plan and what the earlier sub-issues did. Follow the names and contracts they
chose. When the plan is unclear or an earlier sub-issue's work doesn't fit, `epic_status` shows where every sub-issue
stands, and `message_ticket` asks the parent's chat (the coordinator) or a sibling. Ask the architecture expert
before you go to a person.
