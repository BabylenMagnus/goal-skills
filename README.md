# goal-skills

Claude Code skills for the Spelflow goal system ([goal.spelflow.ru](https://goal.spelflow.ru)).

## Skills

| Skill | Trigger | What it does |
|---|---|---|
| `goal` | phrases like "add a goal", "what am I waiting for", "close G-3" | Create, read, edit, close, postpone and delete goals through the goal REST API. Confirms every write with the user first |

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

In Spelflow, Settings → API tokens → Create token, copy the `hly_...` value (shown once). The skill looks for it in this order: environment variable `SPELFLOW_TOKEN` (`SPELFLOW_API_TOKEN` also works), then `.env` in the project root, then the file `~/.spelflow/token`. If none exists it asks you to paste it and offers to save it.

```bash
export SPELFLOW_TOKEN=hly_...        # bash
$env:SPELFLOW_TOKEN = "hly_..."      # PowerShell
```

Keep the token out of git: never commit `.env` or the token file.

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
