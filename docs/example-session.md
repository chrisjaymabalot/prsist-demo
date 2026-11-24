# One session, before and after

This is a made-up Claude Code session in an invented repo, `github.com/example/tide-tables`, and the notes prsist would keep from it. The person and the project are invented. Real sessions never leave prsist, and the transcripts it does receive are deleted within 24 hours.

## Before: what the hook sends

At the end of a session, or when it goes idle, prsist's hook on the user's machine reads the new part of the transcript. It drops tool output, file contents, diffs and the notes prsist itself gave the session at the start. It replaces anything that looks like a secret, then sends what's left to prsist's server.

```text
User: The NOAA API has no UK stations, so the tide times are just wrong for
      Brighton. Let's switch to the Admiralty UK Tidal API. Key is in .env:
      ADMIRALTY_API_KEY=[scrubbed]

Claude: I'll switch the client to the Admiralty UK Tidal API. Since you write
        in UK English, I'll keep the error messages that way too.
        [tool output removed by the hook]
        The client now calls Admiralty. Should I add Redis for caching?

User: No Redis, that's overkill. The free tier only allows 10,000 calls a
      month, so cache responses in SQLite for 6 hours.

Claude: Done. Responses are cached in SQLite for 6 hours.
        [tool output removed by the hook]

User: Also, store every time in UTC and only convert to Europe/London when we
      display it. The BST change broke the March times last year.

Claude: Updated the model and the template filter. All 14 tests pass.
        [tool output removed by the hook]

User: Great, open a PR. Next I want surf height forecasts, but only after the
      tide pages ship.

Claude: Opened the pull request.
```

## After: what prsist keeps

The worker reads the session with the project's existing notes beside it. It pulls out what the user said or decided, then checks each candidate against the closest saved notes.

| Note | Type | What prsist did |
| --- | --- | --- |
| In tide-tables, tide predictions come from the Admiralty UK Tidal API, not NOAA, because NOAA has no UK stations. | decision | **Replaced** the older note "tide predictions come from the NOAA Tides and Currents API". The old one stays as history, so the user can restore it. |
| In tide-tables, API responses are cached for 6 hours in SQLite to stay under the free tier's 10,000 calls a month. | decision | **Added** |
| In tide-tables, every time is stored in UTC and converted to Europe/London only for display. | decision | **Added** |
| The user plans to add surf height forecasts to tide-tables after the tide pages ship. | plan | **Added** |

## What it left out, and why

| From the session | Why it isn't a note |
| --- | --- |
| The API key | Replaced by `[scrubbed]` on the user's machine, before anything was sent. |
| "Should I add Redis for caching?" | Claude suggested it and the user turned it down. prsist keeps what the user decided (SQLite), not the assistant's suggestions. |
| "Since you write in UK English…" | Claude was repeating a note it got at the start of the session. Saving it again would create a duplicate from prsist's own echo. |
| "All 14 tests pass", "Opened the pull request" | Progress on the work, which is out of date by the next session. |
| "The BST change broke the March times last year" | A one-off incident. The decision it led to, storing times in UTC, is what's worth keeping. |

## The next session

The next time the user opens Claude Code in `tide-tables`, prsist's start-of-session hook gives Claude these four notes together with the user's global preferences. Claude also sees them when it calls `recall`. The [README](../README.md#2-the-next-session-starts-with-them) shows a real Claude Code run against the demo server, which holds exactly these notes.
