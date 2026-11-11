# 11. Build order

Build it in phases. Each phase is useful by itself and ends with checks that must pass before you start the next.
My first useful version took about two days, and the version these docs describe took about a week.

## Phases

```mermaid
flowchart LR
  P0["0. Write the contract"] --> P1["1. Sandbox template<br/>+ reaper"]
  P1 --> P2["2. Console POC<br/>go / no-go"]
  P2 --> P3["3. Build jobs in sandboxes<br/>+ tracker bookkeeping"]
  P3 --> P4["4. Deploy the console<br/>to its own host"]
  P4 --> P5["5. Gate, reviewers, approver<br/>auto-merge off"]
  P5 --> P6["6. Context for agents:<br/>chat, errors, prod snapshots, experts"]
  P6 --> P7["7. Splitting, sweeper,<br/>policy watch"]
  P7 --> P8["8. More than one person"]
  P8 --> P9["9. Scheduled work:<br/>intake reviews, error sweep"]
  P9 --> P10["10. Self-improvement<br/>+ production access log"]
```

### 0. Write the contract first

Before any code, write your own `CONTRACT.md`: the job record, states and stages, outcomes → columns, the hook
payload, the tool schemas, the approver's policy and the sweeper rules. Change the contract first, then the code,
every time. It's the document your agents will build against, and it keeps parallel agent work consistent. Docs
03–10 and 13 here are a starting point.

### 1. Sandbox template and reaper

Build one template VM with your repos, toolchain, full app stack with seed data, and headless Chrome. Script the
build so it can be redone from scratch.

- [ ] A clone boots with the stack ready, and you know how long it takes (mine: ~30s to clone, ~6 min to build the template).
- [ ] A clone without the stack boots, and `start_app` brings the stack up on it.
- [ ] An agent in a clone can drive Chrome against the running app.
- [ ] The clone has no production write credentials.
- [ ] The reaper deletes stale `ephemeral` clones and skips `keep`.

### 2. Console POC: go / no-go

A local console that runs jobs in a temp directory, with a chat UI. These checks decide whether the approach works
at all. Run them against the real runtime, driving the console over HTTP/SSE the way the UI does.

| # | Check | Pass means |
|---|---|---|
| 1 | Two jobs in parallel | Separate chats, no crossed events |
| 2a | Steer | A message mid-run is picked up and changes what the agent does |
| 2b | Interrupt | Stops a running tool cleanly; the next message continues the session |
| 3 | Project config loads | The repo's skills, subagents, MCP servers and instruction file are active in the job |
| 4 | Billing proof | `apiKeySource` is recorded and matches what you intend; the guard refuses the wrong setup before any process starts |
| 5 | Restart | Kill the console mid-tool, restart, and the job comes back resumable; the agent remembers a code word from before |
| 6 | Remote transport | The SDK round-trips its stream over ssh to a sandbox, with no relay process needed |
| 7 | Console sends the next turn | The console, not a person, sends a message into an idle session and the agent acts on it (the gate and reviews depend on this) |

Two lessons from my run: a killed console leaves its agent child running until the current tool returns, so store
the pid and stop it on startup (matching on argv so you don't kill an unrelated process). Also close unanswered
permission requests from the dead run.

**If billing fails or the terms say no,** skip to the fallback mode or use API keys. Nothing else changes.

### 3. Build jobs in sandboxes, and tracker bookkeeping

One job kind, `build`, with intake and the builder. Both agent columns start it. Until phase 5, `to_review` moves the
ticket straight to To Review.

- [ ] Hook → job → clone → intake → build → outcome → tracker comment and move → sandbox deleted, end to end on a
  real ticket.
- [ ] Duplicate webhook deliveries create one job.
- [ ] Idle parking: the WIP is pushed and verified, the sandbox deleted, and a message resumes on a fresh clone (mine resumed in ~15s).
- [ ] The 8h cap and the 3-attempt rule both land in Needs Human with a comment.

### 4. Deploy

The console goes on its own long-lived host: UI behind a login proxy, one public port for signed hooks only, local
jobs off, secrets in one mode-600 env file, releases as `releases/<sha>` with a `current` symlink, and an
idempotent deploy script that ships the committed HEAD.

### 5. The gate, reviewers and approver

See [The pipeline](13-pipeline.md). Build them in this order, each on top of the last:

- [ ] **Gate.** `to_review` moves the job to stage `gate`. Red CI hands the builder the log; a new push re-arms it;
  the caps land in Needs Human.
- [ ] **Reviewers.** One prompt per lens, run as subagents after a green gate. `report_review` is refused outside the
  review stage. Changes go back to the builder and through the gate again.
- [ ] **Approver, with auto-merge off.** The policy in code, then the model. Every ticket lands in To Review with the
  approver's view on its PR.
- [ ] **Merge and deploy watch.** Turn auto-merge on for one test repo and merge one low-risk change end to end. A
  red deploy turns the repo back off.

Leave auto-merge off for your real repos at the end of this phase. Read the approver's comments for a few weeks.
Turn a repo on when you'd have merged what it said to merge.

### 6. Context for agents

Chat search and threads, error-tracker reads, scrubbed prod snapshots, and the expert subagents. This is the phase
where intake goes from *"tried some things, couldn't figure it out"* to answers you can act on.

### 7. Splitting, sweeper, policy watch

Plans and epics, umbrella sorts, nesting with a depth cap, the sweeper and the policy watch. See
[Epics and umbrellas](07-epics-and-umbrellas.md) and [Scheduled work](09-scheduled-work.md).

### 8. More than one person

Owners, per-person subscriptions and connections, the setup gate. See [The human side](08-humans.md).

### 9. Scheduled work

Intake reviews for PRs a person opened (the same reviewers, once their CI is green), the daily error sweep, and
anything you were doing with separate scheduled agents. Each becomes a job kind with the same sandbox, tools and
outcomes. Add the Schedules page so it all stays visible.

### 10. Self-improvement and production access

Once jobs run every day, grade them. Stamp each job with its prompt versions first, then add counted facts, then
three rubrics, then the daily improver and the Self-improvement page. See [Self-improvement](12-self-improvement.md).
In the same phase, put every production tool behind a person's approval and one access log
([Production tools need a person](06-agent-tools-and-prompts.md#production-tools-need-a-person)).

## Using agents to build it

I built most of this with agents, and the same rules apply:

- **Land the shared contract serially** (types, routes, schema, `CONTRACT.md`), then fan out with each agent owning
  a disjoint set of files.
- **Verify every delegated result yourself.** Read the diff, run the check. A self-report is not evidence.
- **The POC checks are your regression suite.** Keep them runnable (`pnpm check` or similar) and run them after
  any change to the manager or transport.

## What I'd do differently

| Lesson | What happened |
|---|---|
| **Start with experts and the writing rules, not add them later** | Early results were long, hedged, and parked on "a person must decide". Experts, the tldr/ask format and the "before you report" self-check fixed more than any model change. |
| **Put the strongest model on intake and the reviewers** | Intake is mostly judgment, and a weak reviewer sends good code back. A cheap model in either place makes everything downstream worse. |
| **CI is the test runner, not the sandbox** | Full suites in a sandbox were slow and hit tool timeouts. The builder runs only the tests it touched, pushes early, and lets the gate read CI. |
| **Let code read CI** | An agent watching its own CI will sometimes call a red run green, or stop watching. A tick in the console can't. |
| **Advance epics on Done, not To Review** | Children have to build on *merged* work, or the next child starts from a branch without its predecessor. |
| **Detect umbrellas when the job starts, not when the webhook arrives** | A parent created first has no sub-issues yet when its own event arrives. |
| **Credentials out of the sandbox and off disk early** | Moving tracker, chat and code-host tokens behind a credential proxy left only the agent token and hook secrets on the console host. |
| **Sign every comment** | Once agents and the console wrote as a person's account, nobody could tell who said what. |
| **Make it multi-user from the start** | Retrofitting owners, per-person billing and the setup gate touched most of the manager. |
| **Back up from day one** | The DB is the only copy of every transcript. |
| **Keep company specifics in config and prompt overlays** | Mine ended up hardcoded in many files. If you might share yours, use one config file and base-plus-overlay prompts. |
