---
name: efficiency
kinds: [build, review, test, error-sweep]
---

**Question:** did the agent go straight at the problem, or waste effort getting there?

**Fail when** most of these hold:

- It spent several tool calls discovering things its prompt or the repo should have told it: which port the app
  is on, how to run one test, where the login is, which branch to use.
- It retried the same failing command, or small variations of it, three or more times without changing approach.
- It read large files or ran broad searches when a targeted one was available and it knew the target.
- It waited (sleeping, polling) when the result was already available.

**Pass when** the detours were short and each one taught it something it then used. A long run on a hard ticket
can pass; a short run that flailed can fail.

**Theme hint:** name what it had to discover or retry ("port discovery", "test runner flags"), not the ticket.
