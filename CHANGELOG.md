# Changelog

## 1.0.1 (2026-10-05)

Fixes from review. Behaviour changes:

- `states_ai` is stricter. It no longer treats a reply that only mentions an AI word as an admission
  ("it's a bot-free zone", "I'm happy to help with AI", "I'm checking the machine"). It needs a
  first-person lead, a few plain filler words and a clean ending. This also closes `ensure_opening`,
  `Disclosure(text=...)` and `audit` accepting such text as a disclosure.
- `denies_ai` is wider: "I'm definitely not a bot", "this isn't a bot", "I'm no bot", "a real live
  human being", "a team of real people", "it's Sam from the desk, a real human". `ensure_opening`
  now replaces an opening that claims to be a person instead of putting the notice in front of it.
- `Disclosure(on_silence=...)` and `botcandor answer --on-silence` now default to `"replace"` (was
  `"prepend"`). Pass `"prepend"` for the old behaviour.
- The log's trim check is shared by every `DisclosureLog` on a file in a process, and estimates the
  row count from the file size and its last 64 KB; the whole file is read only when a trim is due.
  A log object built per request no longer reads the whole file on every append.
- A trim keeps the log file's permissions (it was reset to 0600).
- `verify` no longer flags a bare run of digits (such as a millisecond timestamp) as a phone number,
  and takes `surfaces` / `kinds` (`--surface`, `--kind`) for custom values.
- `examples/transcript.jsonl` gains an unanswered question, so it has one of each problem.
- README: the file size bound is stated as `cap` to `cap + 2 * slack` rows.

## 1.0.0 (2026-10-05)

First release.

- `botcandor.DisclosureLog`: an append-only JSON Lines proof log that your AI told people it was an
  AI. A row holds `ts`, `surface`, `session_id`, `kind`, `model`, `prompt_version` and (only when
  true) `forced`, and nothing else. One atomic `O_APPEND` write per row, a ring cap with slack that
  trims by atomic rename, and a `record` that never raises. `read` skips unreadable lines and can cut
  off by date.
- `botcandor.is_identity_question`: finds "are you a real person?", "is this a bot?" and "am I
  talking to an AI?" without catching "I want to speak to a person".
- `botcandor.states_ai` and `botcandor.denies_ai`: read a reply for a first-person admission or
  denial.
- `botcandor.Disclosure`: `answer` keeps an honest reply, repairs or replaces a silent or denying
  one (`on_silence`), and logs the impression; `greeting` makes sure the first line of a call says it
  is an AI (`ensure_opening`); `badge` logs a badge render.
- `botcandor.summarize`, `coverage` and `coverage_pct`: counts by surface, kind and day, and coverage
  against a denominator you count, clamped at 100 and `None` rather than 0% when nothing was logged.
- `botcandor.audit_transcript` and `verify_log`: check a JSON Lines transcript for identity questions
  answered dishonestly, and a log file for fields, surfaces or session ids that should not be there.
- `botcandor` command: `check`, `reply`, `answer`, `record`, `report`, `verify`, `audit`.
- `examples/chat_handler.py` and `examples/transcript.jsonl`.

Limits, stated plainly:

- English only, and pattern-based. It misses "you're a bot, right?" and can fire on "are you a
  computer science major?".
- The log is a plain file. It is append-only by how it is written, not tamper-proof: anyone with
  write access can edit it.
- A row appended while the log is being trimmed can be lost.
- This is not legal advice.
