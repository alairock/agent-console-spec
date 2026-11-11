# Prompts

These are the system prompts my console gives its agents, rewritten as templates. They hold most of the value in
this repo, because each one is the process for its job kind written down. Change them to fit your own process.

## How a job's prompt is built

The console builds each job's system prompt from parts. It never asks the agent to pick its own rules.

```mermaid
flowchart TB
  C[common.md<br/>hard rules, CI, tracker/chat reading,<br/>experts, self-check, writing rules, how to finish]
  K{Job kind}
  C --> P[System prompt]
  K -->|build| B[build.md]
  K -->|build on a parent| U[umbrella.md<br/>in place of build.md]
  K -->|review| R[review.md]
  K -->|test| TE[test.md]
  K -->|error sweep| E[error-sweep.md]
  K -->|grader| GR["grader.md<br/>+ graders/*.md for the kind"]
  K -->|improver| IM[improver.md]
  B & U & R & TE & E & GR & IM --> P
  A{Add-ons} -->|ticket is an epic child| EC[epic-child.md]
  A -->|build job is an epic's final check| EF[epic-final.md]
  EC & EF --> P
  P --> J[Agent job]
  X[experts/*.md + experts/common.md] -->|registered as subagents| J
  RV[reviewers/*.md + reviewers/common.md] -->|registered as subagents| J
  F[First message:<br/>ticket, comments, links, sub-issues] --> J
  M[messages/*.md<br/>gate-red, review-start, review-changes] -->|sent by the console mid-job| J
```

Order: `common.md`, then the kind's file, then any add-ons, then one generated line listing the repos the
sandbox can reach. The first **user** message carries the ticket: title, description, comments with authors (agent
comments marked "(an agent, not a person)"), parent and sub-issues, and for an umbrella the sub-issues to sort.

The experts and reviewers are not part of the system prompt. Each `experts/<name>.md` (its own section followed by
`experts/common.md`) and each `reviewers/<lens>.md` (followed by `reviewers/common.md`) is registered with the agent
runtime as a named subagent, and the job calls it through its normal subagent tool. A build job starts reviewers only
when the console's `review-start` message names them; a review job picks them itself.

## The pipeline's other prompts

A build job runs the [pipeline](../docs/03-pipeline.md). Three more kinds of file drive it:

| File | Who reads it | What it is |
|---|---|---|
| [`reviewers/`](reviewers/) | reviewer subagents | One lens each: `code` (every PR), `security`, `data`, `product`, `domain`. Each ends by calling `report_review`. `domain.md` is a template, like the domain expert |
| [`messages/`](messages/) | the build agent | What the console sends mid-job: `gate-red` (CI failed, with log tails), `review-start` (CI green, start these reviewers), `review-changes` (the findings to fix). `{{slot}}`s are filled by the console |
| [`approver.md`](approver.md) | a one-shot model call, not an agent | The approver's prompt. The console fills in the ticket, risk, policy result, reviews, CI and diff, and reads back `{decision, reason}`. It runs only after the policy in code has passed |

## Frontmatter

| Field | Meaning |
|---|---|
| `name` | Job kind, expert or reviewer name |
| `description` | One line; for experts this is what the calling agent reads to choose one, so write it as "ask it before…" |
| `model` | Model for this kind; omit to use the runtime default. I use the strongest model for build, reviewers, the approver and experts, because that's where judgment lives |
| `disallowedTools` | Tools removed for this kind, e.g. no file edits for a read-only sort, no browser for review |
| `system` | Approver only: the system prompt for the one-shot call |
| `tools` | Experts and reviewers only: an allowlist. Both are read-only; reviewers add `report_review` |

## Placeholders

Fill these in from your config when the prompt is built, or replace them once by hand.

| Placeholder | Example |
|---|---|
| `{{owner}}` | The person the job acts for (whoever moved the ticket) |
| `{{admin}}` | The person who owns the console and approves anything paid or risky |
| `{{org}}` | Your code-host org |
| `{{repos}}` | The repos checked out in the sandbox, with their paths |
| `{{product}}` | One sentence on what your product is and who uses it |
| `{{base_branch}}` | An epic's feature branch, e.g. `feature/ENG-123` |
| `{{app_urls}}` | Where the app runs inside the sandbox, e.g. API `:8080`, frontend `:3000` |
| `{{test_login}}` | Where a throwaway login for the sandbox's local app lives |
| `{{quick_checks}}` | Your fast per-repo checks (fmt, vet, lint, typecheck) |
| `{{targeted_test}}` | How to run a single named test |
| `{{risky}}` | What your team counts as risky: migrations, auth and permissions, billing, deleting data, your core business rules… High risk, and kept out of auto-merge |
| `{{identifier}}`, `{{title}}`, `{{ticket}}`, `{{risk}}`, `{{policy}}`, `{{reviews}}`, `{{ci}}`, `{{diffstat}}`, `{{diff}}` | Approver: filled by the console for each PR |
| `{{round}}`, `{{max}}`, `{{failures}}`, `{{reviewers}}`, `{{prs}}`, `{{sha}}`, `{{findings}}` | Messages: filled by the console when it sends one |
| `{{min_failures}}` | Improver: how many failures a theme needs before it's worth a prompt edit, e.g. 3 |

## The tools these prompts mention

All are served by the console itself as an MCP server named `console`. See
[docs/07](../docs/07-agent-tools-and-prompts.md) for the full contract.

`report_outcome`, `report_review`, `report_wip`, `attach_screenshots`, `tracker_issue`, `tracker_file`, `create_ticket`,
`update_ticket`, `chat_search`, `chat_thread`, `errors_list`, `error_issue`, `error_resolve`, `file_error_ticket`,
`start_app`, `load_snapshot`, `fetch_prod_documents`, `prod_query`, `propose_prompt_edit`.

`load_snapshot`, `fetch_prod_documents` and `prod_query` reach production, so each call waits for a person's approval.

## Graders

`grader.md` is the frame and output format. Each file in `graders/` is one rubric with a `kinds` list in its
frontmatter, and the console gives the grader the rubrics for the finished job's kind. Rubrics are versioned like
prompts, because editing one changes the scores. See [docs/13](../docs/13-self-improvement.md).
