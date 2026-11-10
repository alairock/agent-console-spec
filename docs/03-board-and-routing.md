# 3. Board and routing

The tracker's board is the console's input and its output. Moving a ticket into an agent column starts a job, and
the job's outcome decides which column the ticket lands in next.

## Why

- **The board is the interface people already use.** Nobody learns a new tool to hand work to an agent. They drag
  a card.
- **One front door.** Every new ticket goes to Triage. Triage investigates, then fixes it, plans it, sends it to the
  Lab, or asks. People never have to pick the right agent.
- **Agents judge, the console writes.** An agent reports one structured outcome, and the console turns it into
  comments, moves and sub-issues. Code does the same thing every time. An agent writing to the tracker directly
  would drift, double-post and race with other jobs.
- **People decide what agents work on.** Agents can file and edit other tickets, but they can never put a ticket
  into an agent column. Every unit of agent work starts with a human move.

## Contract

### Columns

| Column | Who acts | What a person sees there |
|---|---|---|
| **Agent Queue** | Triage job | Nothing yet; a job starts within seconds |
| **Agent Lab** | Lab job | Proof the bug happened, then proof it's fixed |
| **In Progress** | A job is running | A comment linking to the live job chat |
| **Plan Review** | You | A plan, with one sub-issue per PR, to approve or edit |
| **Needs Human** | You | One clear question with a recommendation, or "here's what I tried" |
| **To Review** | You | A PR with proof: tests, CI on the pushed SHA, before/after |
| **Done** | – | Merged |

Column names are configuration, not code.

```mermaid
stateDiagram-v2
  state "Agent Queue" as AQ
  state "In Progress" as IP
  state "Agent Lab" as AL
  state "Plan Review" as PR
  state "Needs Human" as NH
  state "To Review" as TR
  [*] --> AQ : a person queues it
  AQ --> IP : job starts
  IP --> TR : PR open with proof
  IP --> AL : triage says it needs the running app
  IP --> PR : needs more than one PR
  IP --> NH : a question only a person can answer, or risky
  AL --> IP : lab job starts
  PR --> AQ : a person approves (runs as an epic)
  NH --> AQ : a person answers and requeues
  TR --> Done : a person merges
  Done --> [*]
```

### Triage routing

Triage investigates first: the ticket, comments, screenshots, linked tickets and threads, the code end to end, and
the data when the bug lives in data. Then it applies the **first** rule that matches:

```mermaid
flowchart TD
  S([Investigation done]) --> Q1{Still unclear, or needs a<br/>product / domain / schema decision?}
  Q1 -->|yes| EX[Ask the matching expert]
  EX -->|expert decides| Q2
  EX -->|expert says escalate| NH1[needs_human<br/>question + expert's recommendation]
  Q1 -->|no| Q2{Risky?<br/>prod data migration, auth/permissions,<br/>billing, paid benchmarks}
  Q2 -->|yes| NH2[needs_human<br/>the risk + the plan you'd follow]
  Q2 -->|no| Q3{More than one PR?}
  Q3 -->|yes| PL[plan_review<br/>one sub-issue per PR]
  Q3 -->|no| Q4{Reproducible with a unit test,<br/>no running app?}
  Q4 -->|yes| FX[Fix it: failing test first, PR, CI green<br/>to_review]
  Q4 -->|no| LB[agent_lab<br/>handoff note for the Lab]
```

The order matters. "Risky" beats "small": a one-line change to an auth check still goes to a person.

### Outcomes

An agent calls `report_outcome` exactly once. After that turn ends, the console applies the row, ends the session
and deletes the sandbox. The job itself ends `done` whatever the outcome; the column carries the meaning.

| Job kind | Outcome | Column | The console also |
|---|---|---|---|
| triage | `to_review` | To Review | comments the PR links |
| triage | `plan_review` | Plan Review | creates `sub_issues` as children; comments the plan |
| triage | `agent_lab` | Agent Lab | comments the handoff note; the move starts a Lab job |
| triage | `needs_human` | Needs Human | comments the questions or the risk |
| umbrella | `umbrella` | In Progress | routes each sub-issue (see [Epics and umbrellas](07-epics-and-umbrellas.md)) |
| lab | `to_review` | To Review | comments the PR links and proof; uploads screenshots |
| lab | `attempt_failed` | (no move) | `attempts += 1`; below 3, tells the agent to start a fresh attempt; at 3, applies `needs_human` |
| lab | `needs_human` | Needs Human | comments why |
| review | `to_review` / `needs_human` | To Review / Needs Human | – |
| any | job crashed, hit the 8h cap, park failed, or ended with no outcome | Needs Human | comments why |

When a job first reaches `running`, the console comments a link to the job's chat and moves the ticket to In Progress.

Every comment has the same shape: a one or two sentence `tldr`, an optional single yes/no `ask`, and the
`summary` folded under "Details". See [The human side](08-humans.md).

### Job kinds

| Kind | Started by | Sandbox | Does |
|---|---|---|---|
| `triage` | Ticket into Agent Queue | No app stack (can start it on demand) | Investigates, routes, fixes small logic bugs |
| `lab` | Ticket into Agent Lab | Full stack + browser | Reproduces in the browser, failing test, fix, independent verifier, PR |
| `review` | A button on a PR, or the PR sync | No stack | Adversarial review; fixes Critical/High on the branch; one findings comment |
| `test` | A button on a ticket | Full stack on the ticket's branches | Brings the app up for a person to try; waits |
| `umbrella` | Triage on a parent with open sub-issues | No stack | Sorts sub-issues into agent / agent_draft / human |
| `error-sweep` | Daily schedule | No stack | Picks up to 3 real errors from the error tracker and files tickets |

### Who is speaking

The console writes to the tracker with one account, so every comment would look like it came from the same person.
Every comment therefore ends with a signature line naming its speaker:

| Speaker | Signature |
|---|---|
| The console's own bookkeeping | `— Agent Console · triage job [job_01…](link)` or `— Agent Console` |
| An agent's own words (its outcome) | `— Triage agent · job […]`, `— Lab agent · job […]`, `— Sort agent · job […]` |
| An interactive agent session | `— Claude (<whose session>)` |
| A person typing in the console | unsigned |

The console parses the line back out. The ticket page shows those comments with an agent author, colored by kind,
and links to the job. Agents reading a ticket see the author as `<name> (an agent, not a person)`, so they don't
treat another agent's guess as a human decision.

With per-person tracker credentials ([The human side](08-humans.md)), the account is the job's owner, and the
signature still says it was an agent.

### Inbound hook

The relay turns tracker webhooks into one call:

```
POST /hooks/tracker-event
X-Console-Signature: hex(hmac_sha256(raw_body, HOOK_SECRET))
{ event, issue_id, identifier, title, url, parent_id, to_state, from_state_id }
```

- `event` is "moved into status" or "created in status". Both are handled the same way.
- The HMAC is over the **raw body bytes**, before JSON parsing. A bad signature returns `401`. A missing secret
  returns `503`, so the hook fails closed.
- `to_state` Agent Queue starts a triage job; Agent Lab starts a lab job. A sub-issue moving to Done is also sent
  (for epics and umbrellas). Anything else returns `202 {ignored: true}`.
- **Idempotent per `(issue_id, to_state)`** while a job for that key is live (any non-terminal state, and
  `needs_human` counts as live). A duplicate returns `200 {job, duplicate: true}`. A new job returns `201 {job}`.
  Relays redeliver, and the console's own moves echo back through the hook. Both must be no-ops.
- `from_state_id` is resolved to a column name. Plan Review → Agent Queue on a ticket with children is an approved
  epic, not a new triage.
- The job's owner is whoever moved the ticket: from the console, or from the tracker's issue history.

### Follow-up chats

A person can message a job after it reported its outcome. That resumes the same session (a follow-up run). When
they press End on a follow-up run:

1. The console asks the agent for a short summary of what changed since the outcome.
2. It posts the reply on the ticket as "Follow-up after the outcome", signed as the agent. A reply of exactly
   `NOTHING` posts nothing.
3. The job ends. The ticket stays where the outcome put it.

Pressing End a second time while the summary is being written ends the job without one. Ending a job that never
reported an outcome sends its ticket to Needs Human.

### Agent writes to other tickets

Agents get `create_ticket` and `update_ticket` for **other** tickets: a follow-up, a bug found on the way, a
comment on a related ticket. Rules:

- Their own ticket moves only through `report_outcome`.
- Agent Queue and Agent Lab are refused as target states.
- New tickets land in Backlog, assigned to the job's owner and linked as related. Max 10 per job.
- Every write shows in the job's chat.

## Acceptance checks

- Moving a ticket into Agent Queue creates exactly one triage job, and the hook returns `201`.
- Replaying the same signed body returns `200 {duplicate: true}` and creates nothing.
- A body with one byte changed returns `401`. With the secret unset, the hook returns `503`.
- A move into any other column returns `202 {ignored: true}`.
- Each outcome in the table produces the listed column and comment, and the comment carries the right signature.
- A Lab job that reports `attempt_failed` three times ends in Needs Human with all three attempts summarized.
- `update_ticket` with state Agent Queue is refused, and the refusal shows in the chat.
- Ending a follow-up run posts the summary. Ending it with a `NOTHING` reply posts nothing.
- The console's own move to In Progress does not create a second job.

## What I used

- **Linear** for the tracker. The webhook goes through a small relay service that was already running, so I only
  needed to fix its payload format. The console can just as well take the tracker's webhook directly.
- Column names come from env vars, so renaming a column is a config change.
- One Linear API key for the console's own writes at first. Per-person tracker credentials came later.
