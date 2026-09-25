# The ask_trace and poll_trace turn protocol

Read this when a turn behaves in a way the skill does not cover: an unexpected status, a
chat you did not mean to continue, an empty result, or a report that will not settle.

## Result shape

`ask_trace` and `poll_trace` return the same shape: one text block of XML-tagged sections,
plus structured content.

- `<ask-trace-turn chatId="..." status="...">` wraps everything.
- `<messages-for-user>` - the chat's replies, in markdown, each possibly with a suggested
  follow-up or an attached report. They are addressed to the user: relay them as they
  arrive.
- `<questions-for-user>` - clarifying questions with numbered options, marked when several
  apply. Relay them verbatim; only the user answers them.
- `<reports>` - report versions no reply introduced.
- `<report reportId="..." documentId="..." title="...">` - a report body. When this client
  already received that exact version, the body is replaced by a pointer to
  `fetch_report`. That is not an error or a truncation.
- `<running>` - what is still in flight, one line each with elapsed seconds. Analysis
  lines carry a `reportId`.
- `<progress-updates>` - present only when a running report's progress moved during this
  call: one line per report with its `reportId`, headline and current step. Tell the user
  in a sentence where each has got to.
- `<relay-instructions>` - what to do next, written for you.

Structured content repeats `chatId`, `turnId`, `status`, the running reports with their
ids and progress, and an advisory `followUp` delay. You have no timer, so calling `poll_trace` is how
you wait.

## Statuses

| Status | Meaning | What you do |
| --- | --- | --- |
| `WORKING` | something is still running | relay `messages-for-user` now, mention `progress-updates` in a sentence, then call `poll_trace` with the same `chatId`, unless the running list is empty; the wait itself is not an answer |
| `INPUT_REQUIRED` | the chat asked a question | relay `messages-for-user` and `questions-for-user` verbatim, then `ask_trace` with the user's own answer as `userMessage`; never answer for them |
| `COMPLETED` | nothing left running | relay everything in full |
| `FAILED` | the message could not be processed | tell the user, offer to retry |
| `CANCELLED` | stopped before it was answered | relay the reason, offer to retry |

`WORKING` with an empty running list is the one case where calling again is pointless.

## Which chat a call lands in

`poll_trace` and `fetch_report` always name their `chatId`. For `ask_trace`:

- `chatId` given: that chat. A malformed, missing or other user's id is refused.
- `newChat: true` and no `chatId`: a fresh chat.
- Neither: the most recent chat this client used, if active within the last half hour,
  else a new one.

A new chat must carry a `userMessage`. Only one turn per chat runs at a time; a second call
is refused while one waits, so retry rather than starting another chat. If another client
sends input into the same chat, your turn ends early.

## Wait budgets

The server holds `ask_trace` and `fetch_report` open for up to thirty seconds, long enough
for the coordinator's first reply, and `poll_trace` for up to sixty. Both sit inside the
point at which hosts cut tool calls off. `waitSeconds` only shortens them.

When the budget runs out you get an ordinary successful result, not an error: whatever
landed, the running lines, and usually still `WORKING`. Nothing is lost, and the next
`poll_trace` resumes where this call stopped. A dropped connection loses nothing either.

## Reports

A `reportId` names one analysis for its whole life. `fetch_report` returns at once if the
report has settled; otherwise it waits briefly, returns the report's progress, and
`poll_trace` is how you wait for it. It takes no lock, so several clients can follow the
same report and following one does not stop you calling `ask_trace`.

`INPUT_REQUIRED` on a report means the analyst has a question, but it asks the chat, not
this call: read it with `poll_trace` on the `chatId`, relay it verbatim, send the user's own
answer with `ask_trace` as `userMessage`, then resume with `poll_trace`.

A `fetch_report` result always carries the report version in full, never the
already-delivered shorthand, plus its assets and the progress steps the analyst declared.
