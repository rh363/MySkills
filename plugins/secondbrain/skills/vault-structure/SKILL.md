---
name: vault-structure
description: >
  This skill should be used whenever the user is working in their Obsidian
  second brain vault and questions arise about where notes belong, how to
  name files, what frontmatter to use, or how the temporal archive
  (Daily/Weekly/Monthly/Yearly) is organized. Trigger phrases include
  "dove va questa nota", "struttura del vault", "convenzioni del second
  brain", "rollup", "archivio", or any operation involving the
  Daily/Weekly/Monthly/Yearly folders. Other secondbrain skills
  (daily-teardown, daily-standup, rollup) reference this skill as the
  source of truth for vault layout. The `session-teardown` skill (mid-day
  per-session checkpoint into `## Work log`) also depends on this skill.
metadata:
  version: "0.3.0"
---

# Vault Structure

Authoritative reference for Alex's Obsidian second brain. Other secondbrain skills depend on the conventions documented here. Do not invent paths or naming schemes — follow this document.

## Vault root

`~/Documents/Obsidian Vault/` (already mounted in Cowork).

## Directory layout

```
/
├── CLAUDE.md           # Instructions for Claude — read every session
├── README.md           # Vault index
├── MOCs/               # Maps of Content (`MOC - <Topic>.md`)
├── Projects/           # One project per folder, kebab-case names
│   └── <kebab-name>/
│       ├── README.md       # Project index (must include alias)
│       ├── Decisions/      # Optional ADR-style notes
│       └── Notes/          # Optional dated notes
├── Areas/              # Cross-cutting knowledge, kebab-case files
├── People/             # Contacts and people
├── Inbox/              # Quick capture, to be triaged
├── Daily/              # YYYY-MM-DD.md — current day work log
├── Weekly/             # In-progress and recently closed weeks
├── Monthly/            # Closed months
└── Yearly/             # Closed years
```

The temporal archive (Daily / Weekly / Monthly / Yearly) is the focus of this plugin. The other folders are pre-existing and follow the conventions in `CLAUDE.md` at the vault root.

## Temporal archive — progressive cascade

A daily note lives in **exactly one** location at any time. The teardown skill moves it through the cascade as periods close.

```
Daily/YYYY-MM-DD.md
    ↓ (evening teardown)
Weekly/MM-DD/YYYY-MM-DD.md           # MM-DD = monday-friday range, e.g. 05-04 means "May 4 to May 8"
    ↓ (end-of-week detected by teardown)
Monthly/MM/MM-DD/YYYY-MM-DD.md       # MM = numeric month, 01..12
    ↓ (end-of-month)
Yearly/YYYY/MM/MM-DD/YYYY-MM-DD.md
```

### Naming rules

- **Daily file**: `YYYY-MM-DD.md` (zero-padded). Example: `2026-05-08.md`.
- **Week folder**: `MM-DD` where MM-DD is the `monday-friday` range expressed as `<start_day>-<end_day>`. Examples:
  - Week of Mon May 4 → Fri May 8 → folder `04-08`
  - Week of Mon May 11 → Fri May 15 → folder `11-15`
  - The range spans the working week (Mon → Fri). Weekends are written in the previous week's folder.
- **Month folder**: zero-padded `MM` (e.g. `05`).
- **Year folder**: 4-digit `YYYY` (e.g. `2026`).

### Computing the week folder name

For a given date `D`:

1. Find the Monday of D's week (if D is Saturday/Sunday, use the Monday that just passed).
2. Find the Friday of D's week (Monday + 4 days).
3. Folder name = `{monday.day:02d}-{friday.day:02d}`.

Python example:

```python
from datetime import date, timedelta

def week_folder(d: date) -> str:
    # weekday(): Mon=0..Sun=6
    monday = d - timedelta(days=d.weekday())
    friday = monday + timedelta(days=4)
    return f"{monday.day:02d}-{friday.day:02d}"

week_folder(date(2026, 5, 8))   # '04-08'
week_folder(date(2026, 5, 11))  # '11-15'
```

> Edge case: when a week straddles two months (e.g., Mon Apr 27 → Fri May 1), the week folder still uses the Monday's day and Friday's day (`27-01`). The skill `rollup` keeps the folder under the **Monday's month** during cascading to Monthly.

## Daily file structure

A complete daily note (after teardown) looks like this:

```markdown
---
created: 2026-05-08
updated: 2026-05-08
tags: [daily]
---

# 2026-05-08 (Friday)

## Schedule
<!-- written by daily-standup in the morning -->
<!-- every line is a checkbox; when the task is finished it's moved into ## Done today as `- [x] ~~...~~` (both checked AND struck-through) -->
- [ ] 09:00–11:00 — Task A
- [ ] 11:00–12:00 — Task B

## Work log
<!-- free-form notes written during the day -->
<!-- each entry MUST be prefixed by its local time in HH:MM 24h format -->
<!-- canonical form: `- **HH:MM — <short title>**: <body>` (single bullet, time bolded inside) -->
- **10:00 — Keycloak / AOP staging**: Daniele ha sbloccato il discorso della **raggiungibilità** della VM. Mandata mail ad **Antonello** per fissare la **call** di test integrazione `aopdev` ↔ `oidc-admin-api`. In attesa di risposta. (#project/keycloak-oidc)
- **11:30 — Refactor billing service**: Estratto `InvoiceCalculator` come port. Restano da migrare 2 use case. (#project/billing)

## Done today
<!-- written by daily-teardown — also: any line in ## Schedule that gets completed during the day is moved here -->
<!-- completed items are BOTH checked AND struck-through, so they read as "definitely closed" -->
- [x] ~~Fix bug X (#project/cloud-firewall)~~
- [x] ~~Review PR ermes (#project/ermes)~~

## Blocked
<!-- written by daily-teardown -->
- ...

## Retro
<!-- written by daily-teardown -->
- ...

## Planned tasks
<!-- planned tasks for THIS day, written by previous day's teardown -->
<!-- IMPORTANT: planned tasks are written WITHOUT checkboxes — plain bullets only. -->
<!-- They acquire `[ ]` only when daily-standup promotes them into ## Schedule. -->
- Task X (#project/cloud-firewall)
- Task Y
```

The same file is written progressively across the day:

1. **Morning** — `daily-standup` reads `## Planned tasks` (plain bullets, no checkbox), prompts Alex, writes `## Schedule` with `- [ ]` checkboxes.
2. **During the day** — Alex writes free-form notes in `## Work log`. Every entry is prefixed by its local time `HH:MM` (24h). When a Schedule line is completed it moves to `## Done today` as `- [x] ~~...~~` (checkbox checked AND text struck-through).
3. **Between sessions** — `session-teardown` appends a timestamped bullet under `## Work log` summarising the chat that just ended, before Alex opens a fresh conversation. May run multiple times.
4. **Evening** — `daily-teardown` reads recently-modified vault files, finalises `## Done today` (checking + striking through), fills `## Blocked` and `## Retro`, then creates/updates the **next working day's** daily with new `## Planned tasks` (no checkboxes), then runs the rollup.

## Frontmatter conventions (vault-wide)

All notes in `Projects/`, `Areas/`, `MOCs/`, and the temporal archive use YAML frontmatter:

```yaml
---
created: YYYY-MM-DD
updated: YYYY-MM-DD
tags: [tag1, tag2]
---
```

For project README files (`Projects/<name>/README.md`), additional fields are required:

```yaml
project: <kebab-name>
aliases: [<kebab-name>, <Display Name>]
status: active | paused | done | idea
stack: [python, go, ...]
```

The `aliases` field is mandatory because Obsidian cannot resolve `[[<project-name>]]` to a `README.md` file otherwise.

## Tag stack (used across daily/teardown/standup)

Project/topic tags follow `CLAUDE.md`. The daily workflow uses these in addition:

- `#daily` — applied to all daily notes
- `#standup` — applied when the standup skill writes `## Schedule`
- `#teardown` — applied when teardown completes the day
- `#planned` — used inline on planned tasks when cross-referencing
- `#blocked` — task or topic that needs external input
- `#todo` — pending action

## What this skill provides

When triggered, this skill:

1. Tells Claude where to write/read each kind of note.
2. Resolves ambiguities about week/month/year folder names.
3. Provides a deterministic algorithm for `week_folder(date)`.
4. Documents the daily note section template.

It does NOT perform actions on the filesystem — that is the job of `daily-teardown`, `daily-standup`, and `rollup`. This skill is consulted as a reference.

## Reading order for related skills

When `daily-teardown`, `daily-standup`, or `rollup` triggers, also load this skill if it is not already in context.
