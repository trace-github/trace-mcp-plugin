# The ask_trace_analyst and poll_trace turn protocol

Read this when a turn behaves in a way the skill does not cover: an unexpected status, a
chat you did not mean to continue, an empty result, or a report that will not settle.

## Result shape

`ask_trace_analyst` and `poll_trace` return the same shape: one text block of XML-tagged sections,
plus structured content.

- `<ask-trace-analyst-turn chatId="..." status="...">` wraps everything.
- `<messages-for-user>` - the chat's replies, in markdown, each possibly with a suggested
  follow-up or an attached report. They are addressed to the user: relay them as they
  arrive.
- `<open-question>` - the question the chat is waiting on the user to answer, with numbered
  options, marked when several apply. It is repeated on every result until the user answers
  it or the chat withdraws it. Relay it verbatim, once; only the user answers it.
- `<reports>` - report versions no reply introduced.
- `<report reportId="..." documentId="..." title="...">` - a pointer to a report, never its
  text. Call `fetch_report` with the `chatId` and `reportId` to show one not yet shown. One
  marked as already delivered was handed to you before; that is not an error or a
  truncation.
- `<running>` - what is still in flight, one line each with elapsed seconds. Analysis
  lines carry a `reportId`.
- `<progress-updates>` - present only when a running report's progress moved during this
  call: one line per report with its `reportId`, headline and current step. Tell the user
  in a sentence where each has got to.
- `<relay-instructions>` - what to do next, written for you.

Structured content carries `chatId`, `turnId`, `status` and `digest`, which is the same text
verbatim, for hosts that show it in place of the text. You have no timer, so calling
`poll_trace` is how you wait.

## Statuses

| Status | Meaning | What you do |
| --- | --- | --- |
| `WORKING` | something is still running | relay `messages-for-user` now, mention `progress-updates` in a sentence, then call `poll_trace` with the same `chatId`, unless the running list is empty; the wait itself is not an answer |
| `WAITING_FOR_USER` | nothing is running; the chat is waiting on the user's answer | relay `messages-for-user` and `open-question` verbatim, then `ask_trace_analyst` with the user's own answer as `userMessage`; never answer for them, never re-ask it in a new chat |
| `IDLE` | nothing is running and no question is waiting | relay every message in full, then `fetch_report` each report not yet shown and relay it in full |

The status is the chat's, read from the chat as a whole, not from what this call happened to
read: a question asked several calls ago is still `WAITING_FOR_USER`. A chat never completes;
it takes the next message. A result marked as an error either refuses the call, naming its
cause (a malformed, missing or other user's `chatId`, an invalid parameter, a turn still
running), or reports that the user's message could not be processed or was stopped: tell
the user, relay any reason, offer to retry.

`WORKING` with an empty running list is the one case where calling again is pointless.

## Which chat a call lands in

`poll_trace` and `fetch_report` always name their `chatId`. For `ask_trace_analyst`:

- `chatId` given: that chat. A malformed, missing or other user's id is refused.
- `newChat: true` and no `chatId`: a fresh chat. Pass it only when the user asks for one.
- Neither: the most recent chat this client used, if active within the last half hour,
  else a new one.

A new chat must carry a `userMessage`. Only one turn per chat runs at a time; a second call
is refused while one waits, so retry rather than starting another chat. If another client
sends input into the same chat, your turn ends early.

## Wait budgets

The server holds `ask_trace_analyst` and `fetch_report` open for up to thirty seconds, long enough
for the coordinator's first reply, and `poll_trace` for up to sixty. Both sit inside the
point at which hosts cut tool calls off. `waitSeconds` only shortens them. A `poll_trace`
on a chat with nothing running returns at once.

When the budget runs out you get an ordinary successful result, not an error: whatever
landed, the running lines, and usually still `WORKING`. Nothing is lost, and the next
`poll_trace` resumes where this call stopped.

## Reports

A `reportId` names one analysis for its whole life. `fetch_report` returns at once if the
report has settled; otherwise it waits briefly, returns the report's progress, and
`poll_trace` is how you wait for it. It takes no lock, so several clients can follow the
same report and following one does not stop you calling `ask_trace_analyst`.

`INPUT_REQUIRED` on a report means the analyst has a question, but it asks the chat, not
this call: read it with `poll_trace` on the `chatId`, relay it verbatim, send the user's own
answer with `ask_trace_analyst` as `userMessage`, then resume with `poll_trace`.

Once the report is complete, a `fetch_report` result carries that version in full, never
the already-delivered pointer, plus the progress steps the analyst declared.
Before then it carries the report's progress.
