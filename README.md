# LectureLens
A simple browser app that can transcribe audio/video files into text, and then anaylse the whole lecture, and create a spreadhsheet of it. There is a built in AI chat bot that runs on Gemini 3.5, which you can talk to if something is not understandable. 

# LectureLens (Python)

Turn a recorded university lecture into a one-sheet study summary: timestamped notes, a "What will be on the exam" section, a glossary, flashcards and practice questions with spaced repetition, and a study assistant you can chat with. Every note links back to the moment in the audio it came from.

This is the Python version: FastAPI + SQLite + plain HTML/JavaScript, with no Node.js and no build step.

---

## Start it (Windows, beginner steps)

**One-time setup**
1. Install **Python** from https://www.python.org/downloads/. On the first installer screen, tick **"Add python.exe to PATH"**.
2. Install **ffmpeg** (it handles audio). Open *Command Prompt* and run `winget install ffmpeg`, then close the window.
3. Unzip `lecturelens-python.zip`.

**Every time**
- Double-click **`start.bat`** in the `lecturelens-python` folder.
  - It runs Python from the **`pyenv`** folder in the project root. If that folder doesn't exist, the first start creates it there (a virtual environment) and installs what the app needs, which takes a minute or two. A `pyenv` folder you made yourself works too: either a virtual environment (`python -m venv pyenv`) or a portable Python unpacked into it.
  - Your browser opens **http://localhost:8000** automatically.
  - Keep the black window open while you use the app. Close it (or press Ctrl+C) to stop.

**Add your free Gemini key** (for real transcripts, notes and chat)
1. Get a key at https://aistudio.google.com → **Get API key**. Don't set up billing; the free tier is enough.
2. Open the file **`.env`** in the folder with Notepad. It is created on the first start.
3. Paste your key after `GEMINI_API_KEY=` with no spaces or quotes, then save.
4. Close the black window and double-click `start.bat` again.

The left sidebar shows which service each part is using, for example "Transcription: gemini · Notes: gemini · Chat: gemini".

**Coming from the earlier JavaScript version?** Copy `db\dev.db` and the `storage` folder from the old folder into this one. Your lectures, notes, flashcards and chats carry over; the database format is identical.

On macOS or Linux, run `./start.sh` instead of `start.bat`.

---

## Which service does what

`auto` (the default) picks the best service you have a key for:

| Part | Order of preference with `auto` | Notes |
|---|---|---|
| Transcription | OpenAI Whisper → **Gemini** → mock | `local` = free offline transcription on your computer (`pip install faster-whisper`) |
| Notes, exam focus, flashcards | Claude → **Gemini** → mock | Claude gives the best notes (paid, a few cents per lecture) |
| Chat assistant | Claude → **Gemini** → mock | streams answers with clickable timestamps |

`mock` is an offline demo. It **ignores your audio** and shows a sample physics lecture, the notes come from a simple keyword heuristic, and the chat can only quiz you from practice questions and search the transcript.

You can force a choice in `.env`, for example `CHAT_PROVIDER=gemini` or `TRANSCRIPTION_PROVIDER=local`.

| Key | Where | Cost |
|---|---|---|
| `GEMINI_API_KEY` | aistudio.google.com | free tier, with daily and per-minute limits. A Gemini app subscription does *not* include API access. Free-tier content may be used by Google to improve its products. |
| `ANTHROPIC_API_KEY` | console.anthropic.com | paid credits. A Claude Pro subscription does not include API access. |
| `OPENAI_API_KEY` | platform.openai.com | about $0.36 per hour of audio |

Gemini notes on transcription: timestamps are approximate (usually within a few seconds), and the "unclear" markers are Gemini's own judgement rather than a measured score. If you see "model not found", set `GEMINI_MODEL` in `.env` to a model listed in AI Studio. When the free limit is hit, the app waits and retries by itself.

---

## Using the app

- **Library (`/`)**: upload MP3/M4A/WAV up to 3 hours (drag and drop), with title, date, course tags and vocabulary hints. Search covers titles, courses, terms and exam topics; sort by date, course or title.
- **Lecture page**: audio player (speed control, section markers) and tabs: Overview · One-Sheet · Timestamps · Exam Focus · Glossary · Full Transcript. Every timestamp is clickable. Fix transcription mistakes with **Edit**; the fix updates the notes, glossary and flashcards immediately, and **Regenerate analysis** rebuilds the notes from the corrected transcript.
- **Exam Focus**: predicted topics with the evidence behind each one. Rate yourself (confident / shaky / lost). After the real exam, mark whether each topic came up; future predictions for the course use those answers.
- **Assistant**: ask questions, say "quiz me", "explain more simply", "make flashcards" (with an **Add to deck** button) or "study plan". It can cover this lecture, the course, or all lectures.
- **Print / PDF**: a one-page A4 sheet; choose "Save as PDF" in the print dialog. **Export**: Markdown, Obsidian, or an Anki import file.
- **Review (`/review`)**: due flashcards and practice questions, scheduled with FSRS. Lost and shaky topics come first.
- **Courses & exams (`/courses`)**: exam dates with a countdown, a day-by-day study plan, and past exams (paste text or upload `.txt`/`.md`).

**Keyboard:** Space play/pause · ←/→ skip 5 s (Shift = 30 s) · `[` / `]` speed · arrow keys move between tabs and confidence buttons · in Review, Space reveals and 1–4 grade.

---

## Debug logs

The black app window shows everything LectureLens does, live. Each line has a time, a colour-coded level and a colour-coded **[category]**:

```
08:38:24.566 INFO    [transcribe] piece 1/4 (0–600 s) → gemini (4.8 MB)
08:38:51.102 INFO    [transcribe] piece 1/4 (0–600 s) ✓ 94 segments, 3 unclear, 8211 chars in 26.5 s
08:39:40.010 WARNING [llm] Gemini API error 429 after 1.2 s (attempt 1/4): quota exceeded
08:41:02.333 ERROR   [browser.error] uncaught: TypeError … at lecture.js:212:9
```

| Category | What it covers |
|---|---|
| `app`, `config`, `server` | start-up, settings, which service is used for what and why, key status (never the key itself) |
| `http`, `page` | every request (status, size, time) and what each page showed |
| `upload`, `audio` | uploads, ffmpeg commands and timings, splitting into pieces |
| `jobs` | background jobs: queued, claimed, finished, retried, failed |
| `transcribe`, `analyze`, `llm`, `cache`, `cost` | every AI request/reply with token counts and timings, validation errors and retries, cache hits, spend |
| `chat`, `chat.offline` | chat messages, context sizes, time to first words, replies |
| `review`, `correct`, `link`, `exam`, `export` | grading (old → new schedule), transcript corrections, see-also links, past-exam matches, exports |
| `db`, `sql` | every SQL statement with row counts and timings |
| `browser.*` | what the web page does (clicks, player, tabs, chat, uploads) **and any JavaScript error** |

Everything is also written, without colours, to **`logs/lecturelens.log`**. Send that file along when reporting a problem. The browser's developer console (F12) shows the `browser.*` lines too.

Settings in `.env`:
- `LOG_LEVEL`: `TRACE` (everything, including idle polling, each streamed chat chunk and raw AI replies), `DEBUG` (default), `INFO` (main events only) or `WARNING`
- `LOG_SQL=false` hides SQL statements.
- `LOG_BROWSER=false` stops forwarding page events.
- `LOG_COLOR=always|never` forces colours on or off.
- `LOG_FILE=` (empty) turns off the log file.

## How it works

```
upload → ffmpeg (mono 16 kHz MP3) → job: transcribe (10-min pieces, cached per piece)
       → "Ready to analyze" (cost shown) → job: analyze (per 45-min part, JSON schema,
         validated, 1 retry, merged, condensed, cached) → flashcards · questions · links
```

- **Jobs (`lecturelens/jobs.py`)**: stored in SQLite and run by a background thread. Claims are atomic, failures retry with exponential backoff, and errors that won't fix themselves (missing key, refusal) fail immediately. Jobs are idempotent.
- **Transcription (`lecturelens/transcription/`)**: a `TranscriptionProvider` interface with OpenAI Whisper (`verbose_json`, word timestamps, confidence from `avg_logprob`/`no_speech_prob`/`compression_ratio`), Gemini (inline audio + JSON schema), local faster-whisper, and a mock. Vocabulary hints are passed to every provider.
- **Analysis (`lecturelens/analysis/`, `lecturelens/prompts/`)**: one structured-output call per part, with Claude via `messages.parse` + Pydantic, or Gemini with a JSON schema. A second validation pass checks every timestamp parses and falls inside the lecture, and that every exam topic has evidence. Timestamps are snapped to segment starts. On failure the errors go back to the model for **one retry**. Long lectures are merged deterministically, then condensed so the one-sheet still fits on a page. The cache key is `sha256(corrected transcript, prompt versions, model, course context)`, so re-running an unchanged lecture costs nothing. Prompts live in versioned files; bump the version when you change one.
- **Chat (`lecturelens/chat/`)**: streams replies. Context is the transcript with your corrections, the notes, confidence ratings and exam countdowns; with Claude, the large part is prompt-cached. History is saved per lecture. Citations like `[12:34]` become seek buttons.
- **Review (`lecturelens/review/`)**: FSRS-4.5 scheduling (capped so items come up again before the exam date), with flashcards matched by term so regenerating keeps your review history. The study plan repeats lost topics 3×, shaky 2×, and confident topics only in the final review.
- **Cost**: shown before analysis runs, and actual spend is tracked per lecture. Prices are in `lecturelens/config.py`.

## Project layout

```
run.py, start.bat, start.sh     start the app
lecturelens/web.py              FastAPI routes (pages + JSON API)
lecturelens/templates, static/  HTML templates, CSS, browser JavaScript
lecturelens/transcription/      OpenAI, Gemini, local faster-whisper, mock
lecturelens/analysis/           schema + validation, chunking, merge, cache, cost, LLM calls, mock
lecturelens/prompts/            versioned prompts (analysis.v1, condense.v1, chat.v1)
lecturelens/chat/               context, Claude/Gemini streaming, offline quiz
lecturelens/review/             FSRS, review queue, sync, study plan
lecturelens/linking/, exam/, export/, corrections.py, pipeline.py, jobs.py, db.py
lecturelens/schema.sql          database tables (same as the JavaScript version)
tests/                          unit + integration tests
eval/                           4 labelled lectures, scorer, runner
```

## Tests and evaluation (for developers)

```bash
python -m pip install -r requirements-dev.txt
python -m pytest
```

There are 58 tests. Unit tests cover timestamps, schema validation, chunk merging, FSRS, cache keys, cost, corrections, linking, past-exam matching, the study plan, and Gemini transcription/analysis/chat against a fake Google client. The integration test runs the real web API end to end: upload → chunked transcription → analysis with one forced validation retry → cache hit → correction propagation → re-analysis → review → exports → courses → offline quiz chat → Gemini chat streaming → job retry.

```bash
python -m eval.run_eval                       # offline baseline: P 0.83 · R 0.61 · F1 0.65 · traceability 1.00
python -m eval.run_eval --provider gemini     # with your free key
```

The eval scores predicted exam topics against hand-checked labels in `eval/lectures/*.json` and saves results to `eval/results/`.

## Limitations

- Anki export is a tab-separated import file, not `.apkg`.
- Past exams accept text, `.txt` or `.md` only; PDF import is not built.
- PDF export goes through the browser's print dialog.
- The player shows a progress bar with section markers, not a waveform.
- Dark theme only.
- Single user, with no login.
- Not built (v2): folder watching, embeddings, study-group sharing.
- The live Gemini, Claude and OpenAI calls are written against the official SDKs and tested with fake clients. A real Gemini transcription already worked in the JavaScript version with the same model.
