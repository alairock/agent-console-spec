---
name: instruction-adherence
kinds: [triage, lab, review, test, umbrella, error-sweep]
---

**Question:** did the agent follow its prompt (`common.md`, its kind's file, any add-ons)?

**Fail when** it broke a rule its prompt states, for example: reported without the required proof, ran a full test
suite it was told not to run, asked a person something an expert should have decided, put more than one question
in `ask`, or touched something it was told to leave alone.

**Also fail** when it followed a rule that was wrong for the situation and that cost it. That's the prompt's
fault, and it's the most useful kind of failure to find.

**The reason must quote the prompt line**, then what the agent did. Without the line, nobody can fix the prompt.

**Pass when** any deviation was the right call for this ticket and the agent said so.
