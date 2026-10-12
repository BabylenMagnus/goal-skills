---
name: goal
description: Use when the user wants to add, look up, change, close, postpone or delete a personal goal in the Spelflow goal system (goal.spelflow.ru) — phrasing like "add a goal", "create a goal for X", "what goals am I working on", "what am I waiting for", "put this goal on hold until Friday", "mark GOAL-3 done", "change the check date", "delete goal GOAL-7". Do not use for ordinary Spelflow Tracker issues (use spelflow-tracker or spelflow-loader) or for anything where the user has not asked to touch goals.
scenarios:
  - "Add a goal: first paying user of Tanuki by the end of November"
  - "What goals are in Waiting and when do I check them?"
  - "Move GOAL-4 to waiting for the access request, check on Friday"
  - "Close GOAL-2, I did it, 100 km done"
license: MIT
allowed-tools: Bash(curl:*)
---

# Spelflow Goals

Work with the user's goals in the Spelflow goal system through its REST API: `https://goal.spelflow.ru/api/goals`. Goals live as issues of the `GOAL` project in the user's Spelflow tracker, so everything you write is also visible there as `GOAL-N`.

## Quick start

```bash
# SPELFLOW_API_TOKEN (hly_...) must already be set in the environment. Never print it.
API="https://goal.spelflow.ru/api/goals"
AUTH="Authorization: Bearer $SPELFLOW_API_TOKEN"

curl -s -H "$AUTH" "$API"                          # all goals
curl -s -H "$AUTH" "$API?status=waiting"           # filter: status, kind, direction
curl -s -H "$AUTH" "$API/GOAL-3"                   # one goal
```

Full field list, statuses, errors and examples: `reference/api-reference.md`. Read it before the first write.

## What a goal is

- **Type:** a `result` (what the user wants to get: weight under 85 kg) or an `action` (a concrete step that serves a result: run 100 km). An action points at its result through `parentId`.
- **Status:** `work`, `waiting`, `later`, `closed`.
- **Waiting needs two things together:** `waitingFor` (what is awaited) and `check` (the date to look again). Never set `waiting` with only one of them. The API rejects it.
- **Direction** (`direction`) is required: the project the goal belongs to (Tanuki, Spelflow, a body goal, and so on).

## Creating a goal

The user should end up with a small, clear goal, not a dumped sentence. Talk it through before writing:

1. Ask what the user wants in their own words.
2. If it is a solution ("buy a treadmill"), ask what result they want from it.
3. Ask how they will know it worked: a number or a clear sign (`measureText`, or a numeric `measure`).
4. For an action, find which result it serves (`parentId`) or say honestly that it serves none.
5. Ask for the first small step (`nextAction`) and, if useful, an if-then trigger (`ifThen`).
6. **Show the finished card and wait for an explicit yes. Only then `POST`.**

Direction and status are never guessed. If the user has not said them, ask.

## Changing a goal

`PATCH` takes only the fields you want to change. A field sent as `null` is cleared. Typical moves:

| User says | Send |
|---|---|
| "wait for X until DATE" | `{"status":"waiting","waitingFor":"X","check":"DATE"}` |
| "back to work" | `{"status":"work","waitingFor":null,"check":null}` |
| "later, return DATE" | `{"status":"later","returnDate":"DATE"}` |
| "done" / "drop it" / "replaced" | `{"status":"closed","closeReason":"done\|cancelled\|replaced","closeNote":"what came out"}` |
| "I weighed 88.2 today" | update `measure.value` and append `{"date":"YYYY-MM-DD","entry":88.2}` to `measure.log` (send the whole `measure`) |

Before moving a goal to `work`, count goals already in `work` (leaf goals only). If there are already three, tell the user and offer to postpone one. This is a warning, not a ban: take it over the limit only if the user says so.

## Deleting a goal

Delete is real and irreversible: the issue disappears from the tracker. Prefer closing (`closeReason: "cancelled"`) unless the user says delete.

1. Show the goal and say it will be erased for good.
2. Wait for an explicit yes.
3. `DELETE /api/goals/GOAL-N`. If the goal has actions the API answers `409` with their ids. Tell the user; use `?force=1` only if they confirm that the actions stay without a result.

## Hard rules

- Write to the API only after the user has seen the exact content and said yes. Reading needs no confirmation.
- Only touch goals the user named or you just listed for them. Never guess a `GOAL-N`.
- Read the token only from `SPELFLOW_API_TOKEN`. Never print it or put it in a goal, a comment or a commit.
- Non-ASCII text (Russian titles): write the JSON body to a file and send it with `curl --data-binary @file`. Inline `curl -d '...'` on Windows corrupts non-ASCII bytes silently, and the request still returns success. After a write, read the goal back and check the text.
- After each write, report back in one line what changed and the goal id.
- Dates are `YYYY-MM-DD`. Never write a date you guessed; ask.
