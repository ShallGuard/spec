# Why write requirements

This page says why a team writes requirements when a coding agent writes
the code. It is for a person who decides whether to adopt ShallGuard.

Developers do not like to write requirements. The reason is simple. The
developer pays the cost now. The benefit comes later, and it comes to
somebody else: an auditor, a reviewer, a future maintainer. So most teams
skip the work, or they write vague documents after the code is done.

Coding agents change this.

When a person wrote the code, a requirement was a document about the
work. Now an agent writes the code, and the requirement is the work.
The requirement is the most important artifact a person produces. It
tells the agent what to build. It tells the reviewer what to check. It
tells the next agent what it must not break.

A prompt does the same job, one time. Then the chat ends and the intent
is gone. A requirement with an anchor does the job every day. The check
runs on every change, forever.

**A specification that you throw away after the code is generated is a
prompt. A specification that a tool enforces forever is a contract.**
ShallGuard is the difference between these two sentences.

You do not need to describe your whole system. Start with the few rules
that you are afraid to lose. Write one SHALL sentence for each rule.
Anchor it to the code and to the test. From that moment, no change breaks
the rule in silence. Not a change from a person, and not a change from an
agent.

The requirement then starts to pay you back:

- The agent follows a written contract, not a memory of a chat.
- The reviewer sees which contracts a change touches, and which tests
  prove that the contracts still hold.
- A new team member reads the requirement document and understands what
  the system promises.
- When something breaks in production, the requirement tells you what
  rule broke, and where the code for that rule is.

Write down what must stay true. The tool keeps it true.
