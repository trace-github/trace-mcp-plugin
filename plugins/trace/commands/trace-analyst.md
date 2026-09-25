---
name: trace-analyst
description: Ask Trace Analyst a question about the business data and drive the answer to completion.
---

Use the `trace-analyst` skill.

Take the question from what the user asked alongside this command and send it in their own
words. If they gave none, ask what they want to know before calling anything.

Add nothing and remove nothing in either direction. Keep the topic in one chat by passing
the same `chatId`. `ask_trace` only passes the user's own words, as `userMessage`: never
call it to wait, and never answer a question on the user's behalf. Relay every message
under `messages-for-user` as it arrives, then keep calling `poll_trace` while the status is
`WORKING` and something is running rather than ending your turn; the wait itself is not an
answer. Relay clarifying questions and their options verbatim, and relay a completed
answer in full.

If the Trace tools are not connected, do `trace-setup` first.
