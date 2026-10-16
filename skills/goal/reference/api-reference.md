# Goal API reference

Base URL: `https://goal.spelflow.ru/api/goals`

Auth: `Authorization: Bearer <token>`. Either a Spelflow Personal API Token (`hly_...`, Settings → API tokens) or the session token of a logged-in Spelflow user. The server asks Spelflow whether it accepts the token and whether the account belongs to the personal workspace; the goals themselves are read from and written to the goal system's own PostgreSQL database. Spelflow Tracker is never written.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/goals` | List goals. Optional filters: `status`, `kind`, `direction` |
| `GET` | `/api/goals/G-N` | One goal |
| `POST` | `/api/goals` | Create a goal. Body: a goal without `id` |
| `PATCH` | `/api/goals/G-N` | Partial update. `null` clears a field |
| `DELETE` | `/api/goals/G-N` | Delete for good. `409` if the goal has actions; `?force=1` overrides |

## Goal object

| Field | Type | Notes |
|---|---|---|
| `id` | string | `G-N`, assigned by the database. Read-only |
| `title` | string | Required. The result or action, as the user would say it |
| `kind` | `"result"` \| `"action"` | Required |
| `direction` | string | Required. Never guessed |
| `status` | `"work"` \| `"waiting"` \| `"later"` \| `"closed"` | Required |
| `parentId` | string | Id of the result this action serves (`G-N`) |
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

## Errors

| Code | Meaning |
|---|---|
| `400` | Bad body: missing title, kind, direction or status; `waiting` without `waitingFor` and `check`; bad id; parent goal not found |
| `401` | No or invalid token |
| `404` | Goal not found |
| `503` | The goal database is not reachable (it sits behind a shared SSH tunnel). Retry later; no data is lost |
| `409` | `DELETE` of a goal that has actions. Body lists them in `children` |
| `502`, `504` | Spelflow (the token check) answered badly or too slowly. Retry once |

## How goals are stored

Each goal is a row of the `goals` table in the goal system's PostgreSQL database, with its tracker links in `goal_links` and the weekly review history in `reviews`. Goals belong to the owner whose token is used. Nothing is written to Spelflow Tracker.
