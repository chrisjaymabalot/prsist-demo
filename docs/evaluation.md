# How prsist's note extraction is evaluated

prsist's value depends on one step: reading a session and keeping the few things worth remembering. If it keeps too little, the next session starts from nothing. If it keeps too much, Claude is fed junk. So prompt changes are measured against sessions I labelled by hand, and the run is repeated after every change.

The sessions are my own, so they stay private. This page gives the method and the overall numbers only.

## The dataset

- **37 real Claude Code sessions** from my own work in November 2026, processed with the hook's own code. Each session goes through the same cleaning, secret scrubbing and chunking as in production, so the run sees exactly what prsist would receive.
- **20 of them labelled by hand so far** (17 in dev, 3 in test), with 65 expected notes across the dev sessions. The run skips the other 17 until they're labelled.
- **Each label file lists:**
  - **expected**: notes prsist must save from this session.
  - **acceptable**: notes it may save without penalty.
  - **prior**: notes that already existed when the session started. The run starts with these saved, so the test can check that prsist doesn't save a duplicate when Claude repeats one.
  - **brief**: the project brief prsist would show the extractor.

  Each note also has a scope (global or project) and a type (preference, decision, plan or fact).
- **A dev/test split.** A hash of the session id sends about one session in three to the test split, so a session never changes side. Prompts are tuned on dev only, and test is kept for the final check.
- **A review flag.** A session counts only once I've checked its whole label file. Two dev sessions were labelled and later left out: they were ones another agent ran through T3 Code, which the production hook no longer sends.

## How a run works

1. Each reviewed session is fed in, chunk by chunk, to the same function the production worker calls. Candidate notes are compared against similar saved notes using the same embedding model as production.
2. A separate judge model (Claude Opus 5.5) matches each expected note to at most one stored note. It labels every stored note expected, acceptable or junk. The judge never sees which prompt version produced the notes.
3. The report gives:
   - **Capture**: the share of expected notes that were saved.
   - **Junk**: the share of saved notes that were neither expected nor acceptable.
   - Each figure's range over repeated runs, and its Wilson 95% bound.
   - The cost of the run.
   - How many saved notes copied one of the prompt's worked examples. This catches the model treating an example as the user's own words.
4. `--sample-for-review` writes random judged notes to a file. I compare them by hand with my labels to check the judge.

The extractor is Claude Haiku 5.5 at medium effort.

## Results on the dev split

| Run | Prompt | Sessions | Repeats | Capture (mean, range) | Junk | Copied examples | Cost of run |
| --- | --- | --- | --- | --- | --- | --- | --- |
| 1 | v5 | 19 | 3 | 61.5% (58.5–63.1%) | 27.5% | 2 | $0.94 |
| 2 | v6 | 17 | 3 | 58.5% (53.8–63.1%) | 21.1% | 0 | $0.81 |
| 3 | v6 | 17 | 3 | **61.5%** (58.5–64.6%) | **22.8%** | **0** | **$0.84** |

How to read this:

- **61.5% capture** in run 3 means prsist saved, on average, 40 of the 65 notes I expected across the 17 dev sessions. The three repeats ranged from 38 to 42 of 65. The Wilson 95% lower bound is 54.5%.
- **$0.84** is the cost of the whole run: 17 sessions, each processed three times. Extraction itself was $0.17. The other $0.67 was the Opus judge, which is used only for evaluation.
- **v5 to v6**: junk fell from about 27% to about 22%, and the prompt stopped copying its own examples (2 to 0). Capture held steady. The runs aren't a strict like-for-like comparison. At the same time I changed the hook to stop sending sessions run by another agent, which removed 2 sessions from dev, and most of the junk drop comes from that. On the same 17 sessions, junk barely moved.
- **Runs 2 and 3** use the same prompt and differ by about 3 points. That spread is why every run repeats each session and reports a range, rather than trusting one number.

## Smaller targeted evals

Some failures are rare but costly, so each has its own eval with many repeats:

- **Reconcile**: notes that only share a topic must never be merged. Two examples are a rail-pass plan and the trip's dates, or a diet and a seat preference. Prompt v4 merged none in 600 runs and caught all 240 true duplicates and updates.
- **Restated notes**: Claude repeating a saved note must not create a new one. 2 failures in 120 runs.
- **Filing**: notes saved from chat apps must land in the right project or stay global. 20 project notes and 25 global ones each run 5 times against 9 projects. In 125 global-note runs, none was filed into a project, and 95 to 100 of the 100 project-note runs landed in the right one.
- **Clustering**: the nightly job that groups loose notes into new projects, such as a trip, must not pull in stable facts about the user.

## What's next

- **Grow the test split** and run the final check on it. Most reviewed sessions are in dev so far, so the numbers above are tuning numbers, not a held-out result.
- **Raise capture without raising junk.** About 4 in 10 expected notes are still missed. The next prompt changes will be judged on whether capture rises while junk stays near 22%.
- **Test prompts on live traffic.** When a prompt changes, run the old and new versions on each queued job before its transcript is deleted, and compare the results.
- **Add opted-in sessions from beta users**, so the set isn't only one person's habits.
