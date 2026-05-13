---
name: session-teardown
description: >
  This skill should be used between coding sessions within a single
  working day, before the end-of-day `daily-teardown`. Trigger phrases
  include "session teardown", "chiudo la sessione", "facciamo il
  teardown della chat", "wrap-up sessione", "checkpoint sessione",
  "salvo la sessione prima di chiuderla", or any request to flush the
  current conversation into the daily note before opening a fresh chat.
  The skill summarises what was done in the current session, asks a
  couple of targeted questions, and appends the result as a timestamped
  entry to today's `## Work log`. It does NOT plan tomorrow and does
  NOT invoke the rollup — those stay with `daily-teardown`. Always load
  vault-structure alongside this skill.
metadata:
  version: "0.2.0"
---

# Session Teardown

Mid-day checkpoint. Captures what happened in the **current chat session** into today's daily note before Alex closes the conversation and opens a new one. This is the per-session counterpart of `daily-teardown` — same vault, same daily file, but lighter and idempotent: you can run it many times in a day.

## Required context

Before doing anything, ensure the `vault-structure` skill is loaded — it defines the daily note path, the section template, and frontmatter conventions. Do not duplicate or contradict that knowledge here.

## When to use this vs. `daily-teardown`

- **`session-teardown`** → between sessions during the day. Does not plan tomorrow. Does not run rollup. Updates `## Work log` only. Can run multiple times.
- **`daily-teardown`** → once at the end of the day. Fills `## Done today` / `## Blocked` / `## Retro`, plans the next working day, and triggers rollup.

If Alex says "chiudo la giornata" / "review serale" / "teardown giornaliero", that's `daily-teardown`, not this skill. If you are unsure which one Alex wants, ask before writing.

## Inputs

- Today's date (use `date +%Y-%m-%d` via Bash if uncertain).
- Current local time (use `date +%H:%M` via Bash) — used as the timestamp prefix of every Work log entry. **This is mandatory** — no Work log entry may be written without an `HH:MM` prefix.
- The vault root: `~/Documents/Obsidian Vault/`.
- The conversation so far — the main signal of what the session did.

## Workflow

Follow these steps in order. Keep it tight: a session teardown should take ~2 minutes, not 10.

### Step 1 — Locate today's daily note

Today's daily lives at `Daily/YYYY-MM-DD.md`. If the file does not exist, create it from the template (see `vault-structure`). If it exists but is missing `## Work log`, append the header at the bottom — never rewrite the file.

### Step 2 — Recap the current session

Form a short mental summary of what was actually done in **this conversation**:

- Files created, edited, or deleted (cite paths).
- Commands run that changed state (commits, migrations, deployments).
- Decisions taken or open questions raised.
- Bugs reproduced / fixed.
- Anything the user explicitly asked you to remember.

Do not pad. If the session was a five-message exchange about a typo, say so.

Optionally cross-check by listing vault files modified since the last session block in today's daily (Bash `find ... -newer <last_block_marker>`), but only if useful — the conversation itself is usually enough.

### Step 3 — Free-form opening (Italian)

Open in Italian. In two or three sentences, summarise what you observed in this session, then ask **one** open question to fill obvious gaps. Examples:

> "In questa sessione abbiamo lavorato sul refactor di X e abbiamo committato Y. C'è qualcosa che vuoi annotare prima di chiudere la chat?"

> "Sessione corta: ho letto i log di Z e abbiamo deciso di rinviare la fix. Aggiungo qualcosa oltre a 'rinviato'?"

Listen to Alex's answer. If Alex says "no, basta così", proceed directly to Step 5.

### Step 4 — Targeted gap-filling (only if needed)

If the session genuinely had loose ends, ask **at most 1–2** further questions. Use AskUserQuestion only when there's a clear set of options. Cover at most:

- **Outcome of the session** — done / paused / blocked.
- **Carryover** — anything that should land in **today's** `## Planned tasks` for a later session today (NOT tomorrow's — that's `daily-teardown`).

Skip this step entirely on quick sessions.

### Step 5 — Append the session entry to `## Work log`

Append a single timestamped bullet (or a small group of bullets, one per topic) under `## Work log`. The canonical form is:

```markdown
- **HH:MM — <short title>**: <one or two sentences. Cite files/paths in backticks. Tag with #project/... if relevant.>
```

If the session covered multiple distinct topics, write one bullet per topic, each with its own `HH:MM` prefix (the time when that piece of work happened, not necessarily "now"). When in doubt, use the current time for all bullets and group them.

Example:

```markdown
- **10:00 — Keycloak / AOP staging**: Daniele ha sbloccato il discorso della **raggiungibilità** della VM. Mandata mail ad **Antonello** per fissare la **call** di test integrazione `aopdev` ↔ `oidc-admin-api`. In attesa di risposta. (#project/keycloak-oidc)
- **11:30 — Refactor billing service**: Estratto `InvoiceCalculator` come port. Restano da migrare 2 use case. (#project/billing)
```

Rules:

- **Every** Work log bullet **must** start with `**HH:MM — <title>**:` (24-hour local time, e.g. `**15:42 — ...**:`). No exceptions. If you cannot determine the time, ask Alex — do not invent it.
- **Always append** to `## Work log` — never overwrite, never reorder existing entries.
- Preserve any free-form notes Alex wrote in `## Work log` himself. Do not refactor them. If you notice an entry of Alex's missing the time prefix, leave it alone — `daily-teardown` will prompt to fix it at end of day.
- If a previous Work log entry has the exact same `HH:MM` prefix and you're adding a separate one, just write a second bullet with the same time — no special discriminator needed; ordering within the same minute is naturally by file order.

### Step 6 — Carryover into today's `## Planned tasks` (optional)

If Alex flagged tasks to pick up later **today** (not tomorrow), append them to today's own `## Planned tasks` section as **plain bullets** (no checkbox — see `vault-structure`):

```markdown
- <Task description> (#carryover)
```

Use the `#carryover` tag so they're distinguishable from tasks planted by yesterday's teardown. If there are no carryovers, skip this step entirely.

### Step 7 — Update frontmatter

In today's daily note frontmatter:

- `updated: <today>` — set to today's date (idempotent if already today).
- Do **not** add `#teardown` to `tags` — that tag is reserved for `daily-teardown`. Leave `tags` untouched unless `#daily` is missing, in which case add it.

### Step 8 — Confirm

Give Alex a one-line wrap-up in Italian: file path updated, time of the entries appended, and any carryover. Example:

> "Aggiunte 2 voci in `## Work log` di `Daily/2026-05-08.md` (15:42), con 1 carryover (#carryover) nei planned di oggi. Pronta per chiudere la chat."

Do **not** invoke `rollup`. Do **not** create or edit any other daily file. The only file you touch is today's `Daily/YYYY-MM-DD.md`.

## Style rules

- Italian by default. Technical terms can stay in English.
- Be brief. A session teardown is a checkpoint, not an interview.
- Never invent content. If the session was trivial, write one bullet and stop — but it still **must** carry the `**HH:MM — ...**:` prefix.
- Always append, never rewrite. Multiple entries in a single day are expected and desirable.
- Never touch tomorrow's daily, never run rollup, never set `#teardown` in frontmatter — those are `daily-teardown`'s responsibilities.
- If Alex follows up with "ok, chiudo la giornata" right after, hand off to `daily-teardown`; do not duplicate its work.
