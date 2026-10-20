# Goal API reference

Base URL: `https://goal.spelflow.ru/api/goals`

Auth: `Authorization: Bearer <token>`. Either a Spelflow Personal API Token (`hly_...`, Settings → API tokens) or the session token of a logged-in Spelflow user. The server asks Spelflow whether it accepts the token and whether the account belongs to the personal workspace; the goals themselves are read from and written to the goal system's own PostgreSQL database. Spelflow Tracker is never written.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/goals` | List goals. Optional filters: `workspace`, `status`, `kind`, `direction` |
| `GET` | `/api/goals/today` | The one main task for today: `{goal, reason}` with `reason` `overdue`, `due-today`, `blocks-others` or `oldest`, or `{goal: null}`. `?workspace` limits the candidates |
| `GET` | `/api/goals/G-N` | One goal |
| `POST` | `/api/goals` | Create a goal. Body: a goal without `id` |
| `PATCH` | `/api/goals/G-N` | Partial update. `null` clears a field. `dependsOn` replaces the whole list |
| `DELETE` | `/api/goals/G-N` | Delete for good. `409` if the goal has actions; `?force=1` overrides |
| `GET` | `/api/goals/G-N/entries` | Measurement journal, latest first. `?limit`, `?voided=1` also shows undone entries |
| `POST` | `/api/goals/G-N/entries` | Record a measurement. Body below. `201`, or `200` with `duplicate: true` for a repeated `key` |
| `DELETE` | `/api/goals/G-N/entries/ID` | Undo an entry (also `POST`). Idempotent |
| `PATCH` | `/api/goals/G-N/entries/ID` | `{value, date, finishedAt}` corrects a live entry; `{voided: false}` restores an undone one |
| `GET` | `/api/goals/G-N/days` | Per-day totals and insights. `?from=YYYY-MM-DD&to=YYYY-MM-DD`, default last 30 days |
| `GET` | `/api/goals/G-N/pace` | Needed per day to the deadline and actual last 7 days |
| `POST` | `/api/goals/G-N/session` | Start a timed session `{source, key?}`. `409` if one is open. `DELETE` cancels it |
| `POST` | `/api/goals/G-N/session/finish` | Close the open session with a value (same body as an entry). `409` if none is open |

## Goal object

| Field | Type | Notes |
|---|---|---|
| `id` | string | `G-N`, assigned by the database. Read-only |
| `title` | string | Required. The result or action, as the user would say it |
| `kind` | `"result"` \| `"action"` | Required |
| `workspace` | string | Required. `me`, `rusal`, `tanuki` or `spelflow` (the keys under `linkWorkspaces` in `GET /api`). The token must see that workspace. Never guessed |
| `direction` | string | Optional subgroup inside a workspace, for example "Body" inside `me` |
| `status` | `"work"` \| `"waiting"` \| `"later"` \| `"closed"` | Required |
| `parentId` | string | Id of the result or action this action serves (`G-N`). Only an action has a parent; a result has none. Same workspace. A chain of actions is allowed, a cycle is not |
| `dependsOn` | string[] | Cards (`G-N`, any workspace) that must be closed before this one can start: "waits for". `[]` or `null` clears. A card cannot wait for itself, for what it serves, or in a cycle. An open (not `closed`) dependency keeps the card out of `GET /api/goals/today` |
| `measureText` | string | Free-text measure while no number is set |
| `measure` | object | `{unit, start, target?, value, mode, ticks?, log}`. `mode` is `"sum"` (entries add up) or `"last"` (entry is the new reading). `log` is `[{date, entry}]` |
| `recurring` | string | Schedule, e.g. "every Sunday" |
| `nextAction` | string | The first small step |
| `ifThen` | string | If-then trigger |
| `waitingFor` | string | Required together with `check` when `status` is `waiting` |
| `check` | `YYYY-MM-DD` | Next check date |
| `returnDate` | `YYYY-MM-DD` | When a `later` goal comes back |
| `deadline` | `YYYY-MM-DD` | |
| `closeReason` | `"done"` \| `"cancelled"` \| `"replaced"` | With `status: "closed"` |
| `closeNote` | string | What came out |
| `links` | array | Optional pointers to a tracker project or task: `[{"workspace": "<slug>", "project"?: "<KEY>", "issue"?: "<KEY-N>", "label"?: "<text>"}]`. `GET /api` lists the workspace slugs. Read-only for the tracker |

Unknown fields are ignored.

## Examples

Create (write the body to a file first if it contains Russian text):

```bash
curl -s -X POST -H "$AUTH" -H "Content-Type: application/json; charset=utf-8" \
  --data-binary @goal.json "https://goal.spelflow.ru/api/goals"
```

`goal.json`:

```json
{
  "title": "Run 100 km by 8 November",
  "kind": "action",
  "workspace": "me",
  "direction": "Body",
  "status": "work",
  "parentId": "G-1",
  "measure": {"unit": "km", "start": 0, "target": 100, "value": 0, "mode": "sum", "log": []},
  "recurring": "three times a week",
  "ifThen": "If it is Monday, Wednesday or Saturday morning, then I run before work",
  "deadline": "2026-11-08"
}
```

Response `201`: `{"goal": {... "id": "G-5"}}`.

Move to waiting:

```bash
curl -s -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"status":"waiting","waitingFor":"access request","check":"2026-10-20"}' \
  "https://goal.spelflow.ru/api/goals/G-4"
```

Back to work (clear the waiting fields):

```bash
curl -s -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"status":"work","waitingFor":null,"check":null}' \
  "https://goal.spelflow.ru/api/goals/G-4"
```

## Entry object and journal calls

`POST /api/goals/G-N/entries` body: `{"value": 2.1, "source": "bot", "key": "tg:123:456", "date": "2026-10-17", "startedAt": "...", "finishedAt": "...", "note": "..."}`. `value` and `source` (`bot`, `site` or `import`) are required; `date` defaults to today in the goal timezone (Europe/Moscow); `finishedAt` is an ISO time. Response: `{"entry": {...}, "goal": {...}, "duplicate": false}`.

| Field | Notes |
|---|---|
| `id` | Entry id; grows with time of creation |
| `value` | Number, three decimals. `null` while a session is open |
| `date` | `YYYY-MM-DD`, the calendar day in the goal timezone |
| `startedAt`, `finishedAt` | ISO times, optional |
| `source` | `bot`, `site`, `import` |
| `note`, `key` | Optional. `key` is unique per goal |
| `voidedAt` | Set when undone. The entry stays in history and leaves the total |

`GET /api/goals/G-N/days` returns `{mode, unit, tz, from, to, today, days: [{date, value, count, first?, last?, change?}], gaps: [...], insights: {activeDays, total?, avgPerActiveDay?, best?, worst?, currentStreak?, longestStreak?, byHour?, peakHour?, min?, max?, change?}}`. For `sum` goals `value` is the day total; for `last` goals it is the last reading and `change` is the difference from the previous reading.

`GET /api/goals/G-N/pace` returns `{"pace": {value, target, unit, remaining, daysLeft, overdue, needPerDay, actualPerDay7, onTrack}}`; `400` if the goal has no measure target or no `deadline`.

## Errors

| Code | Meaning |
|---|---|
| `400` | Bad body (for an entry also: a `sum` value not above zero, a bad date, `finishedAt` before `startedAt`, an empty `PATCH`): missing title, kind, workspace or status; `waiting` without `waitingFor` and `check`; bad id; unknown workspace; parent goal not found, in another workspace, or a result given a parent; a cycle in `parentId` or `dependsOn`; a card waiting for itself or for what it serves; a `dependsOn` card that does not exist or is not the owner's |
| `403` | The token does not see the chosen workspace |
| `401` | No or invalid token |
| `404` | Goal or entry not found (an undone entry cannot be corrected, restore it first) |
| `503` | The goal database is not reachable (it sits behind a shared SSH tunnel). Retry later; no data is lost |
| `409` | `DELETE` of a goal that has actions (body lists them in `children`); a session is already open; no open session to finish |
| `502`, `504` | Spelflow (the token check) answered badly or too slowly. Retry once |

## How goals are stored

Each goal is a row of the `goals` table in the goal system's PostgreSQL database, with its tracker links in `goal_links`, its "waits for" links in `goal_dependencies` (deleting a goal removes its links) and the weekly review history in `reviews`. Goals belong to the owner whose token is used. Nothing is written to Spelflow Tracker.

Progress is not stored in the goal. Every measurement is a row of the `goal_entries` table (`value`, `entry_date`, `started_at`, `finished_at`, `source`, `note`, `dedupe_key`, `voided_at`). `measure.value` and `measure.log` in a goal response are **derived** from the live entries: `start` plus the sum for `mode: sum`, the latest reading for `mode: last`, and the last 20 entries as `[{date, entry}]`. The column `goals.measure` holds only the settings (`unit`, `start`, `target`, `mode`, `ticks`), so a `PATCH` that sends `value` or `log` has no effect. An undone entry keeps its row with `voided_at` set. The same `dedupe_key` per goal never creates a second entry; at most one session (an entry without a value) is open per goal.
