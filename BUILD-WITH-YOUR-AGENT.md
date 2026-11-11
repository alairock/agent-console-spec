# Build your own with your agent

Paste the prompt below into your coding agent, from an empty repo with this spec repo checked out next to it (or
give it the URL). It reads the specs, interviews you about your setup, writes **your** contract first, and then
builds in phases. It won't move to the next phase until the current phase's checks pass. Expect the interview to
take a while: your answers are what make the result yours.

````text
You're going to build an "agent console" for my team. The design is in the agent-console-spec repo (path or URL:
<fill in>). It's a spec, not code. Your job is to build my version of it, shaped around my tools and my process.

## 1. Read, in this order
README.md (glossary included), then docs/01 through docs/13, then prompts/README.md and every file in prompts/.
Don't write any code while reading. When you're done, summarize back to me in 10 bullets: the principles you'll
hold to, and the parts you think I might not need.

## 2. Interview me
Ask me these one group at a time. Wait for my answers before asking the next group. Push back when an answer
conflicts with a principle in docs/01, and tell me which one.

Process
- What does "shipping" mean on my team today, step by step, from ticket to production? Which steps get skipped?
- Which tickets do I want agents to take: bugs only, features, reviews, error-tracker issues?
- What's risky for us (prod data, auth, billing, paid model runs, anything else)? Agents will stop and ask on these.
- Who reviews and merges? Who answers domain questions today, and what do they usually get asked?
- Does a merge to main deploy to production? Which repos, if any, may an approver merge on its own later, and up to
  what risk? Which paths must always go to a person (migrations, auth, billing, infrastructure, anything else)?
- Which review lenses matter here (code, security, data, product, our domain), and when should each run?

Tools
- Tracker: which one, and can I add board columns? Does it send webhooks?
- Code host and CI: which ones? Can CI run our full test suite on a pushed branch?
- Error tracker and team chat: which ones, and should agents read them?
- Agent runtime: which coding agent and SDK? Can it be driven programmatically with streaming input (steer,
  interrupt, approve tools)? How is it billed (subscription or API key), and what do the provider's current terms
  say about driving it from my own app? Quote the terms; don't assume.

Sandboxes
- Where can I get throwaway VMs or containers with an API or CLI to clone, tag, list and delete?
- Can the app's whole stack run in one sandbox? How long does it take to boot? Is there a scrubbed data seed?
- Can a headless browser run there for the agent to drive?
- How do secrets reach a sandbox without being written to its disk?

People
- Is this just for me at first, or for several people? If several, does each person bill their own subscription
  and act under their own name in the tracker and code host?
- How will people reach the console securely (SSO proxy, VPN, tailnet)?

## 3. Write CONTRACT.md first
Before any code, write CONTRACT.md for my version: the job record and its states, job kinds, the outcome table
(outcome, column, what the console does), the pipeline's stages, the approver's policy, the agent tool list with
schemas, the inbound webhook (with signature), the sweeper rules and every limit (concurrency cap, idle park, hard
cap, attempts, gate and review rounds, the auto-merge cap, per-tool caps). Use docs/03, 04, 05, 07 and 10 as the
template and my answers as the content. Show it to me and wait for my approval. From then on,
change CONTRACT.md first and the code second.

## 4. Build in phases (docs/12-build-order.md)
For each phase: say what you'll build, build it, run that phase's go/no-go checks for real (not just unit tests),
and show me the evidence (commands, output, IDs). Don't start the next phase until I've seen the checks pass. If a
check fails, stop and tell me. Don't work around it.

Rules while building
- The console's code does all tracker writes for a job's own ticket. Agents report one structured outcome.
- The console, not an agent, reads CI. Reviewers report their own verdicts. Agents never merge; the console merges
  only under the approver's policy in code, and auto-merge stays off for every repo until I turn it on.
- Every job runs in its own fresh sandbox. Agents never run on the console's host, next to its secrets.
- No secret is written to a sandbox's disk or put on a command line.
- Sandboxes hold no production credentials. Production is reached only through console tools, and each call
  waits for a person and lands in the Production access log.
- Keep it boring: one process, one SQLite file, one UI. No queue service, no microservices, no plugin system.
- Adapt the prompts in prompts/ to my process and fill every placeholder. Write the domain expert by interviewing
  me (or the person who answers domain questions) using the template's questions.
- When you're unsure about something that's mine to decide, ask. When it's a technical default, pick the boring
  option, note it in CONTRACT.md, and keep going.
````
