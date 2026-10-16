# Goal API reference

Base URL: `https://goal.spelflow.ru/api/goals`

Auth: `Authorization: Bearer <token>`. Either a Spelflow Personal API Token (`hly_...`, Settings → API tokens) or the session token of a logged-in Spelflow user. The server forwards your token to the Spelflow tracker, so you see and change exactly what your account can in the `GOAL` project of each configured workspace. Nothing is stored server-side.

## Spaces

A goal lives in the workspace of its project. Spaces are configured on the server; `GET /api` lists them with the directions that map to each. Goals are addressed as `<space>:GOAL-N` (`me:GOAL-1`, `rusal:GOAL-2`) because tracker numbers repeat across workspaces.

| Key | Workspace | Directions |
|---|---|---|
| `me` | Personal ("Личное") | Тело, Отношения |
| `rusal` | Русал | Русал |
| `tanuki` | Тануки | Tanuki |
| `spelflow` | Spelflow | Spelflow |

There is no default space. If a direction maps to none and the body has no `space`, `POST` answers `400` and names the available keys.

## Endpoints

| Method | Path | Purpose |
|---|---|---|
| `GET` | `/api/goals` | List goals from all spaces. Optional filters: `status`, `kind`, `direction`, `space` |
| `GET` | `/api/goals/<space>:GOAL-N` | One goal |
| `POST` | `/api/goals` | Create a goal. Body: a goal without `id`, plus `space` when the direction maps to none. A goal with `parentId` goes to its result's space |
| `PATCH` | `/api/goals/<space>:GOAL-N` | Partial update. `null` clears a field |
| `DELETE` | `/api/goals/<space>:GOAL-N` | Delete for good. `409` if the goal has actions; `?force=1` overrides |

## Goal object

| Field | Type | Notes |
|---|---|---|
| `id` | string | `<space>:GOAL-N`, assigned by the tracker. Read-only |
| `title` | string | Required. The result or action, as the user would say it |
| `kind` | `"result"` \| `"action"` | Required |
| `direction` | string | Required. Never guessed |
| `status` | `"work"` \| `"waiting"` \| `"later"` \| `"closed"` | Required |
| `parentId` | string | Id of the result this action serves (`<space>:GOAL-N`). It must be in the same space |
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
  "parentId": "me:GOAL-1",
  "measure": {"unit": "km", "start": 0, "target": 100, "value": 0, "mode": "sum", "log": []},
  "recurring": "three times a week",
  "ifThen": "If it is Monday, Wednesday or Saturday morning, then I run before work",
  "deadline": "2026-11-08"
}
```

Response `201`: `{"goal": {... "id": "me:GOAL-5"}}`. The space came from the parent. For a new direction add `"space": "rusal"` (or another key) to the body.

Move to waiting:

```bash
curl -s -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"status":"waiting","waitingFor":"access request","check":"2026-10-20"}' \
  "https://goal.spelflow.ru/api/goals/rusal:GOAL-4"
```

Back to work (clear the waiting fields):

```bash
curl -s -X PATCH -H "$AUTH" -H "Content-Type: application/json" \
  -d '{"status":"work","waitingFor":null,"check":null}' \
  "https://goal.spelflow.ru/api/goals/rusal:GOAL-4"
```

## Errors

| Code | Meaning |
|---|---|
| `400` | Bad body: missing title, kind, direction or status; `waiting` without `waitingFor` and `check`; bad id; no space for the direction; parent in another space |
| `401` | No or invalid token |
| `404` | Goal not found |
| `409` | `DELETE` of a goal that has actions (body lists them in `children`), or `POST` into a space whose workspace has no `GOAL` project |
| `502`, `504` | The tracker answered badly or too slowly. Retry once; the tracker can take tens of seconds |

If `GET /api/goals` returns `{"missing": true, "goals": []}`, the `GOAL` project exists in none of the configured workspaces. `missingSpaces` lists the keys of workspaces that have no `GOAL` project yet.

## How goals are stored

Each goal is an issue in the tracker. Fields that an issue does not have are kept as a short readable list at the top of its description, between `<!-- goal:v1 -->` and `<!-- /goal -->`. Text below that list is free text: the API never touches it. The list endpoint of the tracker returns no status, so the status of a goal is read from this block. The tracker status is mirrored on every write (`In Progress`, `Waiting`, `Backlog`, `Done`, `Canceled`).
