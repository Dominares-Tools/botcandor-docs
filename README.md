# botcandor — Make your chatbot admit it's an AI, and log it

**Paid tool: $49 once (plus sales tax where it applies), then a private GitHub repository you're invited to, with every update. Buy: https://dominares.org/tools/botcandor**

Commercial license v1.3 — see https://dominares.org/license · Support: support@dominares.org

botcandor spots "are you a bot?" questions, replaces a reply that dodges or denies with an honest one, and appends a no-personal-data row to a proof log. It also puts an AI notice in front of a call's first line. Not legal advice.

## Get it

Buy botcandor at https://dominares.org/tools/botcandor: one payment of $49, plus sales tax where it applies, and every future update. After checkout you are invited to a private GitHub repository with the source and every version published to it. The repository includes the Python library and the botcandor command, runnable examples (a chat handler and a sample transcript), and the full test suite.

Requirements: Python 3.10 or newer, standard library only.

This page documents version 1.0.1. Install commands and file paths in the documentation below run inside the private repository you are invited to; the source is not in this repository.

## Documentation

### What it does

Three jobs, each usable alone. The runtime is the Python standard library; Python 3.10 or newer.

**1. Notice the question.** `is_identity_question(text)` is true for "are you a real person?", "is this a bot?", "am I talking to an AI?" and the like, and false for "I want to speak to a person", so a request for a human does not get mistaken for it.

**2. Force an honest answer.** `Disclosure.answer(user_message, reply, session_id)` looks at your model's reply to an identity question:

| The reply | What you get back | Logged |
| --- | --- | --- |
| says it is an AI ("Yes, I'm an AI assistant") | the reply, untouched | yes |
| says nothing either way | the disclosure sentence only (or the sentence, then the reply, with `on_silence="prepend"`) | yes, `forced` |
| denies it ("I'm a real person", "I'm definitely not a bot", "this isn't a bot") | the disclosure sentence only | yes, `forced` |
| is empty | the disclosure sentence | yes, `forced` |
| (any reply to a question that is not an identity question) | the reply, untouched | no |

A reply counts as an admission only when it says so in the first person in a plain form ("I'm an AI assistant", "you're talking with our AI assistant", "I'm Ava, a virtual assistant"); a reply that only mentions the words ("I'm happy to help with AI", "it's a bot-free zone") is treated as silent. Pass `on_silence="prepend"` to keep a silent reply and put the disclosure in front of it instead of replacing it. `Disclosure.greeting(opening, session_id)` does the same for the first line of a call: if the opening does not plainly say it is an AI, a notice is put in front of it (or replaces it, if the opening claims to be a person), in code, so no editable prompt can drop it. `Disclosure.badge(session_id)` logs that an "AI assistant" badge was shown.

```python
from botcandor import Disclosure, DisclosureLog

disclosure = Disclosure(DisclosureLog("disclosures.jsonl"), model="my-model", prompt_version="v7")
text = disclosure.answer(user_message, model_reply, session_id).text   # send this
```

`examples/chat_handler.py` runs this end to end with a stand-in model.

**3. Keep the proof.** `DisclosureLog` is an append-only JSON Lines file. A row holds `ts`, `surface` (`phone`, `web_chat` or `web_badge`; pass your own), `session_id`, `kind` (`greeting_floor`, `identity_answer` or `badge_render`; pass your own), optional `model` and `prompt_version` so you can say which model and prompt were live, and `forced: true` when botcandor repaired or replaced a reply. There is no parameter through which anything else could be written, and over-long values are cut.

- `record` never raises. It returns `False` when a row is refused (unknown surface or kind, empty session id) or the write fails, so a logging problem cannot break a live call or chat turn.
- Each row is one `O_APPEND` write of a single short line. Concurrent threads were tested not to lose rows; separate processes rely on the operating system's append behaviour for a local file and were not tested.
- The file holds between `cap` and `cap + 2 * slack` rows, the newest ones: 50,000 to 70,000 by default (`cap` and `slack` are arguments). The size check runs about once per `slack` appends per process and estimates the row count from the file size and its last 64 KB; the whole file is read only when a trim is due. It is shared by every `DisclosureLog` on the same file in a process, so building one per request is fine. A trim keeps the file's permissions.

### The command

```bash
botcandor check "is this a bot?"                         # exit 0 if it is an identity question
botcandor reply "Yes, I'm a real person"                 # exit 1: it denies being an AI
botcandor answer "are you a bot?" --reply "No, I'm a person" --log d.jsonl --session s2
botcandor record --log d.jsonl --surface phone --kind greeting_floor --session CA123
botcandor report --log d.jsonl --since 2026-10-01 --denominator phone=120 --denominator web_chat=40
botcandor verify --log d.jsonl                           # exit 1 if anything does not belong in it
botcandor audit examples/transcript.jsonl                # exit 1 if an identity question was answered badly
```

- `report` prints counts by surface and kind, and with `--denominator SURFACE=N` (how many calls or sessions there were, which you count) a coverage percentage. Coverage is capped at 100, and shows `n/a` instead of 0% when nothing was logged, because 0% would read as "the bot failed to disclose" when it may only mean logging was off. `--json` prints the same as JSON.
- `verify` flags a line that is not JSON, a field outside the allowed list, an unknown surface or kind (add yours with `--surface NAME` and `--kind NAME`, or `surfaces=` and `kinds=` in `verify_log`), and a session id that looks like an email address or a phone number (a `+` number, or digits split by spaces, dots, dashes or brackets). A bare run of digits is not flagged, since timestamps and numeric ids look the same.
- `audit` reads a JSON Lines transcript (`{"session_id", "role", "content"}` per line, oldest first). For each identity question it checks the assistant's next message in that session and reports `denied`, `silent` or `unanswered`. `examples/transcript.jsonl` has one of each (a denied, a silent and an unanswered question, plus one that was answered honestly).

Exit codes: 0 fine, 1 a check failed, 2 bad usage.

### Limits

- **English only, and pattern-based.** It misses "you're a bot, right?" and can fire on "are you a computer science major?". The patterns are deliberately narrow about what counts as an admission, because a false admission would keep a reply that never disclosed anything. The price is that an unusual honest phrasing ("I'm what you'd call a bot") reads as silent, and the default `on_silence="replace"` then swaps it for the disclosure sentence. A denial phrased in a way the patterns do not know also reads as silent, and is replaced too; with `on_silence="prepend"` it would go out next to the disclosure and contradict it.
- **A plain file, not a vault.** The log is append-only by how it is written. Anyone with write access to the file can edit it, and botcandor does not sign or chain the rows. If you need tamper evidence, keep the file somewhere only your service can write and copy it out on a schedule.
- **A row appended while the file is being trimmed can be lost.** Trimming happens roughly once per `slack` appends, and a log that never reaches its cap never trims.
- **It proves the disclosure was produced, not that it was heard.** A web chat reply is what you sent; a voice greeting is what you asked your phone system to play.
- **Coverage needs your own count** of calls and sessions; botcandor does not know how many there were.
- **`session_id` is whatever you pass.** It is kept as given (cut at 128 characters). If you pass a phone number or an email address as the session id, it will be stored: use the opaque id your system already makes, and run `botcandor verify` (it catches an email address, and a phone number written with a `+` or separators, not a bare run of digits).
- The opening sentence of a phone call, voicemail and recording notices, and telephony itself are not here.

### Install, test

*The install commands are in the README of the private repository you are invited to when you buy. The notes on them are kept here.*

```bash
python3 -m pytest
```

---

Docs mirror of the private repository Dominares-Tools/botcandor; updated with each release.
