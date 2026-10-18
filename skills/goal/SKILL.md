---
name: goal
description: Use when the user wants to add, look up, change, close, postpone or delete a goal (personal or work, for example Rusal, Tanuki, Spelflow) in the Spelflow goal system (goal.spelflow.ru), or asks what to do next in a workspace or direction — phrasing like "add a goal", "create a goal for X", "what goals am I working on", "what am I waiting for", "what next", "what are our goals", "how is Rusal going", "what should I do about Tanuki", "put this goal on hold until Friday", "mark G-3 done", "change the check date", "delete goal G-7", or reports or asks about a measurement ("I ran 2.1 km", "I weigh 88", "how many km did I run yesterday", "undo the last entry"). Do not use for ordinary Spelflow Tracker issues (use spelflow-tracker or spelflow-loader).
scenarios:
  - "What do we do next on Rusal?"
  - "How is the Tanuki direction going?"
  - "Add a goal: first paying user of Tanuki by the end of November"
  - "What goals are in Waiting and when do I check them?"
  - "Move G-4 to waiting for the access request, check on Friday"
  - "Close G-2, I did it, 100 km done"
license: MIT
allowed-tools: Bash(curl:*)
---

# Spelflow Goals

Work with the user's goals in the Spelflow goal system through its REST API: `https://goal.spelflow.ru/api/goals`. Goals are their own records in the goal system's database (PostgreSQL). They are **not** tracker issues. A goal is addressed as `G-N` (for example `G-3`) and belongs to one workspace (personal, Rusal, Tanuki, Spelflow). A goal may optionally link to a tracker project or task (see "Links"), and the link is only a pointer: writing a goal never changes anything in the tracker.

## Quick start

```bash
# Token (hly_...): shared convention, same in every Spelflow skill. Never print it.
T="${SPELFLOW_TOKEN:-$SPELFLOW_API_TOKEN}"
[ -z "$T" ] && [ -f .env ] && T=$(grep -E '^SPELFLOW_(API_)?TOKEN=' .env | head -1 | cut -d= -f2- | tr -d '\r\n "')
[ -z "$T" ] && [ -f ~/.spelflow/token ] && T=$(tr -d '\r\n ' < ~/.spelflow/token)
[ -z "$T" ] && { echo NO_TOKEN; exit 1; }
AUTH="Authorization: Bearer $T"
API="https://goal.spelflow.ru/api/goals"

curl -s -H "$AUTH" "$API"                          # all goals
curl -s -H "$AUTH" "$API?workspace=rusal"          # one workspace
curl -s -H "$AUTH" "$API?status=waiting"           # filter: workspace, status, kind, direction
curl -s -H "$AUTH" "$API/G-3"                       # one goal
curl -s https://goal.spelflow.ru/api               # index of the API
```

Token lookup order: environment variable `SPELFLOW_TOKEN` (or `SPELFLOW_API_TOKEN`), then `.env` in the project root, then `~/.spelflow/token`, then a token the user already gave in this conversation. If none is found (`NO_TOKEN`), ask the user to paste it (Spelflow, Settings → API tokens → Create token, shown once) and offer to save it: project `.env` (make sure `.gitignore` lists `.env`), the global file `~/.spelflow/token`, or not at all. Paste the block above at the start of each Bash call that uses the API.

Full field list, statuses, errors and examples: `reference/api-reference.md`. Read it before the first write.

## What a goal is

- **Type:** a `result` (what the user wants to get: weight under 85 kg) or an `action` (a concrete step that serves a result: run 100 km). An action points at its result through `parentId`.
- **Status:** `work`, `waiting`, `later`, `closed`.
- **Waiting needs two things together:** `waitingFor` (what is awaited) and `check` (the date to look again). Never set `waiting` with only one of them. The API rejects it.
- **Workspace** (`workspace`) is required: where the goal lives. The keys are listed by `GET https://goal.spelflow.ru/api` under `linkWorkspaces`: `me` (personal: weight, relationships), `rusal`, `tanuki`, `spelflow`. There is no separate "goals" project or workspace. An action and its result are always in the same workspace.
- **Direction** (`direction`) is optional: a subgroup inside a workspace, for example "Body" and "Relationships" inside `me`. Leave it empty when the workspace is enough (Rusal).
- **Links** (`links`) are optional pointers to a tracker project or task, for example a goal "finish the A0A project" linked to that project. Many goals have none ("Relationships", "Weight"). See "Links".

## "What next?" mode

When the user asks what the goals are, how a direction is going or what to do next, answer from the system first. This mode only reads until the user agrees to a write.

1. **Read first.** `GET` the goals of the named workspace (`?workspace=...`, add `direction` if one is named). The workspace is clear from the question ("Rusal", "my weight") or is the one of the project you work in; propose it, and ask one question only if it is not clear. Never ask "goals in the system or project goals?". If the workspace has no cards, say that in one line ("no cards in <workspace> yet") and do not list goals from other workspaces.
2. **Show the state in at most 5 lines:** goals in `work` (the most important first, at most three lines), goals in `waiting` with check dates, overdue items (check date or deadline passed). State overdue neutrally and offer three moves: shrink the step, move the date, or close it. No judgement.
3. **Two deadlines.** If the context gives two different deadlines for one goal, ask once which one is the goal's deadline. Without a deadline there is no pace.
4. **Only then use project context** (wiki, PM files, what the user told you). Each plan item either points to an existing goal or is offered as a new card. Offer cards only for next actions and waits, at most three at a time. Other plan items stay in the chat or a PM file, so the goal list does not fill up.
5. **External dependencies** (a request, a reply from a person, a ticket): offer a `waiting` card with `waitingFor` and a check date, but only when the request has already been sent. If the question has not been asked yet, offer an action card ("ask X about Y") instead and create the `waiting` card after the question is asked. Never invent the date; ask.
6. **End with one recommended step**, not a choice between options. If the user will do the step themselves, ask the two questions from "Creating a goal", step 5: what will most likely get in the way, and when exactly they start (an event or a time). If the user has a scheduled event (a call, a meeting) that fits, offer it as the trigger for `ifThen`. Skip the questions if an agent does the step.
7. **Say the risk in one line.** If the context gives both a deadline and an amount of work (for example 18 tasks of about a day each and 17 calendar days), state in one line that they may not fit and what is unknown (task size, a blocking dependency). Do not conclude; the user decides.
8. Keep other handles for a short "later" line (at most three items), not a list of equal options.
9. Every write follows the Hard rules: only after the user has seen the exact content and said yes.

## Creating a goal

To answer "what next" without creating anything, see the previous section.

The user should end up with a small, clear goal, not a dumped sentence. Talk it through before writing:

1. Ask what the user wants in their own words.
2. If it is a solution ("buy a treadmill"), ask what result they want from it.
3. Ask how they will know it worked: a number or a clear sign (`measureText`, or a numeric `measure`).
4. For an action, find which result it serves (`parentId`) or say honestly that it serves none.
5. For an action that will start in `work`, ask three short questions, one at a time:
   1. The first small step (`nextAction`).
   2. What will most likely get in the way? One sentence. Build the trigger on this obstacle.
   3. When exactly will they start? The answer becomes `ifThen`: "If <event or time>, then <step>". The "if" must be something that happens by itself (a time, an event, the end of a previous task), not a decision to start. If the answer is only an intention ("when I sit down to work on X"), suggest tying it to an event or time once, and accept the answer if the user insists. Do not reject it.
   The user may skip questions 2 and 3: save the card without `ifThen` and do not press. Do not ask 2 and 3 for a `result`, or for an action that starts in `later`.
6. **Show the finished card and wait for an explicit yes. Only then `POST`.**

Workspace and status are never guessed. If the user has not said them, ask; you may propose the workspace from the project you work in, but the user confirms it.

**One round, one yes.** Collect everything missing in one message (not one question per message), then show the finished card once and wait for one explicit yes. If a date is ambiguous ("two weeks" and "until the 5th" differ), show both dates in the same message and let the user pick.

**An intermediate result.** A result cannot sit under another result (`parentId` is only for actions). If the user wants a step toward a bigger goal, offer two options: an `action` under the bigger goal (when the step is mostly in the user's hands), or rewrite the bigger goal itself (when it was vague and is now concrete). Do not create a neighbour card that is linked only by name.

## Links

A goal is not a task. "Finish the whole A0A project" is a goal and the project is the work behind it; "Weight below 85 kg" is a goal with a number and no project; "Relationships" is a distant goal that has no project at all. So:

1. Never create a goal as a tracker task and never write into the tracker. The goal lives in the goal system's own database.
2. Ask whether the goal relates to an existing tracker project or tasks only when the user mentions one. Do not push a link.
3. A link is `{"workspace": "<workspace slug>", "project": "<key>", "issue": "<KEY-N>", "label": "<optional text>"}`; `project` and `issue` are optional. The workspace slugs are listed by `GET https://goal.spelflow.ru/api` under `linkWorkspaces`.
4. Links are read-only for the system: the goal never changes a task's status or due date.

## Changing a goal

`PATCH` takes only the fields you want to change. A field sent as `null` is cleared. Typical moves:

| User says | Send |
|---|---|
| "wait for X until DATE" | `{"status":"waiting","waitingFor":"X","check":"DATE"}` |
| "back to work" | `{"status":"work","waitingFor":null,"check":null}` |
| "later, return DATE" | `{"status":"later","returnDate":"DATE"}` |
| "done" / "drop it" / "replaced" | `{"status":"closed","closeReason":"done\|cancelled\|replaced","closeNote":"what came out"}` |
| "I weighed 88.2 today", "ran 2.1 km" | not a `PATCH`: record a measurement, see "Recording measurements" |

When moving an action to `work` that has no `ifThen`, offer once to add the obstacle and the trigger (step 5, questions 2 and 3); do not insist.

There is no limit on goals in `work`, neither a warning nor a count. The user decided this on 2026-10-16.

## Recording measurements

A numeric `measure` has `mode`: `sum` (each entry adds to the total, for example km run) or `last` (each entry is the new reading, for example weight). The progress (`measure.value`, `measure.log`) is **derived** from the entry journal. Never send `value` or `log` in a `PATCH`: the server ignores them. Record entries instead:

```bash
# body.json: {"value": 2.1, "source": "site", "key": "optional-unique-key"}   (source: bot | site | import)
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json" --data-binary @body.json "$API/G-2/entries"
```

- Add `"date": "YYYY-MM-DD"` for another day (default: today in Moscow time), `finishedAt` (ISO time) for the time of day, `note` for a comment. Numbers use a dot; the API also accepts a comma in strings.
- A `sum` entry must be greater than zero. A wrong entry is **undone**, not subtracted: `DELETE $API/G-2/entries/ID`. The entry stays in history, leaves the total, and can be restored with `PATCH` and `{"voided":false}`. `PATCH` with `{"value","date","finishedAt"}` corrects a live entry.
- The same `key` never creates a second entry (`200` with `"duplicate": true`). Use a key when a retry is possible.
- History and analysis: `GET $API/G-2/entries?limit=10` (latest entries with time), `GET $API/G-2/days` (per-day totals plus best and worst day, streak, gaps, by hour; `?from=&to=`, default last 30 days), `GET $API/G-2/pace` (needed per day to the deadline, actual last 7 days, `onTrack`). The server computes all of these. Report its numbers; do not recompute.
- Timed sessions: `POST $API/G-2/session` starts one (`{"source":"bot"}`; `409` if one is open), `POST $API/G-2/session/finish` closes it with a value, `DELETE $API/G-2/session` cancels it.
- Record only what the user said they did, in the goal they named. Show the entry first and wait for a yes, like any other write (see Hard rules). The Telegram bot and the site are clients of the same API.

## Deleting a goal

Delete is real and irreversible: the goal disappears from the database. Prefer closing (`closeReason: "cancelled"`) unless the user says delete.

1. Show the goal and say it will be erased for good.
2. Wait for an explicit yes.
3. `DELETE /api/goals/G-N`. If the goal has actions the API answers `409` with their ids. Tell the user; use `?force=1` only if they confirm that the actions stay without a result.

## Hard rules

- Write to the API only after the user has seen the exact content and said yes. Reading needs no confirmation.
- Only touch goals the user named or you just listed for them. Never guess a goal number (`G-N`).
- Find the token only as described in Quick start (the shared Spelflow token convention). Never print it or put it in a goal, a comment, a wiki page or a commit.
- Non-ASCII text (Russian titles): write the JSON body to a file and send it with `curl --data-binary @file`. Inline `curl -d '...'` on Windows corrupts non-ASCII bytes silently, and the request still returns success. After a write, read the goal back and check the text.
- After each write, report back in one line what changed and the goal id.
- Dates are `YYYY-MM-DD`. Never write a date you guessed; ask.
