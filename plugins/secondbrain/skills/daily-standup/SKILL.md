---
name: daily-standup
description: >
  This skill should be used at the start of a working day to plan and
  schedule today's tasks in Alex's Obsidian second brain. Trigger phrases
  include "standup", "facciamo lo standup", "pianifica oggi",
  "buongiorno pianifichiamo", "schedula la giornata", or any morning
  request to set up the day. The skill reads today's daily note (created
  by yesterday's teardown), reviews the planned tasks, asks Alex for
  priorities and time estimates, and writes a `## Schedule` section into
  today's daily note. Always load vault-structure alongside this skill.
metadata:
  version: "0.3.1"
---

# Daily Standup

Morning workflow. Turns yesterday's planned tasks into today's schedule.

## Required context

Ensure `vault-structure` is loaded — it defines the daily-note template, frontmatter, and the `## Schedule` section convention.

## Inputs

- Today's date.
- The vault root: `~/Documents/Obsidian Vault/`.

## Workflow

### Step 1 — Open or create today's daily

Path: `Daily/YYYY-MM-DD.md`.

- If the file exists (likely created by yesterday's teardown), Read it.
- If it does not exist, create it from the template in `vault-structure`. There will be no `## Planned tasks` section in this case — proceed and ask Alex to dictate today's plan from scratch.

### Step 2 — Read planned tasks

Look for the `## Planned tasks` section. Extract each bullet item.

> **Important**: planned tasks are stored as plain bullets (`- Task X`), **without** checkboxes. They acquire `[ ]` only when promoted into `## Schedule` in Step 5. If you find legacy `- [ ]` entries in `## Planned tasks`, treat them as plain bullets and silently normalise them when you rewrite the section.

If the section is missing or empty, ask Alex what they want to do today — collect tasks conversationally.

### Step 3 — Brief recap

In Italian, briefly recap to Alex what was planned. One short paragraph. Example:

> "Da ieri sera hai 4 task pianificati per oggi: [...]. Cominciamo a metterli in ordine?"

### Step 4 — Prioritise and estimate

Walk through the task list with Alex. For each task, gather:

- **Priority** (P0 must-do today / P1 should-do / P2 nice-to-have).
- **Time slot or estimate** — either a clock time (`09:00–11:00`) or a rough duration (`~1h`, `~30min`).

Use AskUserQuestion for priority when there are several tasks of unclear ranking. Otherwise stay conversational. If Alex pushes back on a task ("non oggi"), keep it in `## Planned tasks` but mark it with `#postponed`. Do not silently delete tasks.

If Alex wants to add new tasks not in the plan, accept them. Append them to `## Planned tasks` (as **plain bullets, no checkbox** — see Step 2) with the `#standup` tag, then schedule them.

### Step 5 — Write the schedule

Insert a `## Schedule` section right after the H1 title, before `## Work log`. Every Schedule line **must** be a checkbox `- [ ]` — this is the discriminator vs. `## Planned tasks` (plain bullets). Format:

```markdown
## Schedule

- [ ] 09:00–11:00 — Implementare bridge L2 PoC3 (#project/cloud-firewall) — P0
- [ ] 11:00–12:00 — Review PR ermes (#project/ermes) — P1
- [ ] 14:00–15:30 — Studio Kubernetes networking (#study) — P2
```

Rules:

- Every line is a `- [ ]` checkbox. When the task is completed during the day, the checkbox is flipped **in place** to `- [x]` (plain text — **no strikethrough**, `~~...~~` breaks Obsidian rendering). The line stays in `## Schedule` so the plan you made in the morning remains visible as memory. `## Done today` is reserved for ad-hoc work that was *not* in the schedule.
- Order strictly by P0 → P1 → P2, then by chronological time.
- If Alex did not give clock times, replace the time slot with `~30min` / `~1h` etc.
- Preserve any pre-existing `## Schedule` section if Alex re-runs the standup mid-morning — append a `### Update HH:MM` block instead of overwriting.

Update the frontmatter:

- `updated: <today>`
- Add `#standup` to `tags` if not present.

### Step 6 — Final summary

Confirm in chat what you wrote: number of tasks scheduled, total estimated time, anything postponed. Example:

> "Schedule scritto: 4 task per circa 5 ore. 1 task postponed (#postponed: 'studio kubernetes networking'). Buona giornata!"

## Style rules

- Italian by default.
- Be concise — the standup should take less than 5 minutes of conversation. Don't ask 10 questions when 2 will do.
- Respect what Alex already wrote in the daily; only add the `## Schedule` section.
- If today already has a `## Schedule` (re-run), use the `### Update HH:MM` append pattern instead of replacing.
