# goal-skills

Claude Code skills for the Spelflow goal system ([goal.spelflow.ru](https://goal.spelflow.ru)).

## Skills

| Skill | Trigger | What it does |
|---|---|---|
| `goal` | phrases like "add a goal", "what am I waiting for", "close GOAL-3" | Create, read, edit, close, postpone and delete goals through the goal REST API. Confirms every write with the user first |

## Install

### Claude Code plugin marketplace

```
/plugin marketplace add BabylenMagnus/goal-skills
/plugin install goal-skills@goal-skills
```

### Any agent supported by `npx skills`

```bash
npx skills add BabylenMagnus/goal-skills
```

## Token

In Spelflow, Settings → API tokens → Create token, copy the `hly_...` value (shown once), then set it yourself. Never paste it into a chat:

```bash
export SPELFLOW_API_TOKEN=hly_...        # bash
$env:SPELFLOW_API_TOKEN = "hly_..."      # PowerShell
```

## Layout

```
skills/
  goal/SKILL.md
  goal/reference/api-reference.md
.claude-plugin/
  marketplace.json
  plugin.json
```

API shapes, fields and errors: `skills/goal/reference/api-reference.md`.
