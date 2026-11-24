# Design decisions

The main choices behind prsist, what each one costs, and the alternatives I turned down. Trust is the product, so most of these come down to one question: what is the least prsist can hold and still be useful?

## Capture automatically, through hooks and MCP

**Choice.** Claude Code has hooks, so prsist captures from it automatically. When a session ends or goes idle, a hook sends the new part of the transcript to prsist. When a session starts, another hook gives Claude the relevant notes. Codex has an experimental hooks engine, and hooks for it are planned. Until then Codex, like the Claude apps, which have no hooks, connects through MCP, and the model calls prsist's tools itself.

**Why.** If saving is manual, prsist is just a notes file. MCP is the one interface that Claude, Claude Code and Codex all support, so one server reaches all three with one account.

**Cost.** Capture in the chat apps is best effort. Tool descriptions alone weren't enough to steer Claude.ai, so setup ends with one line the user pastes into their Claude instructions.

## Keep three tools, not ten

**Choice.** `recall`, `remember` and `forget`. Each description says exactly when to call it. Merging, renaming and moving notes happen in the dashboard, not through tools.

**Why.** Models pick tools more reliably from a short list. Each tool is marked read-only or destructive, which Claude's connector directory requires and ChatGPT uses to decide when to ask for confirmation.

## Keep notes, not conversations

**Choice.**
- Transcripts are encrypted with a key that only the app server holds, so the database and its backups store only ciphertext.
- A transcript is deleted as soon as its job finishes. An hourly sweep inside Postgres deletes any transcript still waiting after 23 hours, so none is kept longer than 24 hours.
- Secrets are scrubbed on the user's machine before anything is sent.
- Each note keeps a one-line evidence snippet instead of the transcript.

**Why.** People rate their AI chats as more sensitive than email, and prsist collects them from several tools at once. A privacy promise is only worth something if the system enforces it, so the sweep runs inside the database. It keeps working even if the whole app is down.

**Cost.** Without transcripts, I can't re-run extraction over old sessions after a prompt improves. That's why the [evaluation](evaluation.md) uses my own sessions, labelled by hand, instead of production data.

## Extract on prsist's server, not the user's machine

**Choice.** Distillation runs on prsist's server with Claude Haiku 5.5, a small, fast and cheap model.

**Rejected.** Running extraction locally with the user's own Claude Code or Codex. It would use up the user's AI quota, and every user would get a different model and prompt, so quality couldn't be tested or controlled.

**Cost.** Users have to trust prsist's server with the transcript for a few minutes, which is why the retention rules above matter.

## Two scopes, with the git remote as the project key

**Choice.** Every note is either **global**, such as "writes in UK English", or belongs to **one project**, such as "tide-tables caches responses in SQLite". In coding tools, the git remote URL identifies the project. Folder paths differ between machines, but the remote doesn't. Notes from chat apps, like a trip or a training plan, are filed by meaning. A nightly job groups loose notes on one topic into a new project.

**Why.** A preference should follow you everywhere, but a project fact shouldn't leak into another repo. The notes a session starts with are the current project's first, then global ones, and never another project's. When Claude searches with `recall`, it searches all of the user's notes, closest first, because the user may be asking about another project.

## Update notes instead of piling them up

**Choice.** Each new candidate is compared with the closest saved notes. The model decides to add it, replace an older note, mark it as a duplicate (counting it as seen again), or ignore it. A replaced note is kept as history, so the dashboard can show what replaced it and restore it.

**Why.** Facts change: an API gets swapped, a date moves. If old notes were never replaced, recall would hand Claude contradictions.

**Cost.** A wrong replace hides a true note, which costs more than a duplicate. So the prompt adds the note whenever the model is unsure, and a [targeted eval](evaluation.md#smaller-targeted-evals) checks that notes which only share a topic are never merged.

## Treat recalled notes as data, not instructions

**Choice.** `recall` wraps each note in its own `<note id saved>` element, escapes the text so a note can't close its element, and puts a line first saying the notes are information about the user, not instructions.

**Why.** A web page pasted into one chat could otherwise plant an instruction that steers Claude in every later session. The [demo server](../src/prsist_demo/store.py) frames notes the same way, and has a test for it.

## A small stack, one language

| Part | Choice | Why |
| --- | --- | --- |
| Server | Python, FastAPI with FastMCP | One language and one deploy for the API, the MCP server and the worker |
| Database | Postgres with pgvector (Supabase, London) | Notes, vectors, the job queue and scheduled jobs in one place, with row-level security on every table |
| Worker | A second process working through a job queue in Postgres | Model calls can take a minute, and retries and leases are just rows |
| Dashboard | React, run only in the browser, served as static files | No server-side rendering, so the 2025–26 server-component vulnerabilities don't apply to it |
| Hosting | Fly.io, London | Two small always-on machines: one web, one worker |

## Measure before tuning

Every prompt change is run against hand-labelled sessions, with a separate judge model and repeated runs. Results are reported as ranges, not single numbers. See [evaluation](evaluation.md).
