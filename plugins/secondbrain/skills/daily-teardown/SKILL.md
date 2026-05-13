---
name: daily-teardown
description: >
  This skill should be used at the end of a working day to close out the
  daily note in Alex's Obsidian second brain. Trigger phrases include
  "fai il teardown", "chiudo la giornata", "review serale", "teardown
  giornaliero", "facciamo il wrap-up della giornata", or any request to
  summarise today and plan tomorrow. The skill reads files modified today
  in the vault, runs a free-form interview with targeted questions to
  fill the gaps, completes today's daily note, writes planned tasks into
  the next working day's daily note, and finally invokes the rollup skill
  to archive today's note into the Weekly folder (cascading further if
  end-of-period is detected). Always load vault-structure alongside this
  skill.
metadata:
  version: "0.3.1"
---

# Daily Teardown

End-of-day workflow. Closes today's daily note, plans tomorrow, and runs the archival rollup.

## Required context

Before doing anything, ensure the `vault-structure` skill is loaded — it defines paths, naming, and the daily-note template. Do not duplicate or contradict that knowledge here.

## Inputs

- Today's date (use `date +%Y-%m-%d` via Bash if uncertain).
- The vault root: `~/Documents/Obsidian Vault/`.

## Workflow

Follow these steps in order. Do not skip ahead even if you think you have enough information.

### Step 1 — Locate today's daily note

Today's daily lives at `Daily/YYYY-MM-DD.md`. If the file does not exist, create it from the template (see `vault-structure`). If it exists but is missing the standard sections, do **not** rewrite it — append the missing section headers at the bottom.

### Step 2 — Gather today's vault activity

Find every file in the vault modified today (excluding `.obsidian/` and `.git/`). Use Bash:

```bash
find "/home/alex/Documents/Obsidian Vault" \
  -type f -name "*.md" \
  -not -path "*/.obsidian/*" \
  -not -path "*/.git/*" \
  -newermt "$(date +%Y-%m-%d) 00:00:00" \
  ! -newermt "$(date +%Y-%m-%d) 23:59:59" \
  -printf "%T+ %p\n" | sort
```

For each modified file (excluding the daily note itself), Read it and form a mental summary of what changed in Alex's work today. Pay special attention to:

- New or updated `Projects/<name>/README.md` files.
- ADRs in `Projects/<name>/Decisions/`.
- New notes in any `Notes/` subfolder.
- Anything tagged `#todo`, `#blocked`, `#decision`, `#idea`.

### Step 3 — Free-form opening

Open the conversation in Italian (Alex's preferred language). Briefly summarise — in plain prose, not a list — what you observed today across the vault. Two or three sentences.

Then ask an open question, e.g.:

> "Ho visto che hai lavorato su X e Y. Vuoi raccontarmi com'è andata oggi, o partiamo da qualcosa di specifico?"

Listen to Alex's answer fully before drilling down.

### Step 4 — Targeted gap-filling

After the free-form opening, ask 2–4 mirror questions that close the gaps you still have. Use AskUserQuestion only when there is a clear set of options; otherwise ask conversationally. Cover at minimum:

- **What's done?** — items completed today. Items that were in `## Schedule` get their checkbox flipped to `[x]` in place (stay in Schedule). Ad-hoc work that was not in the schedule goes into `## Done today`.
- **What's blocked?** — items waiting on external input (`## Blocked`). Tag with `#blocked` and the cause.
- **Retro/insights** — anything worth remembering for the future (`## Retro`). Optional.
- **Tomorrow's priorities** — task list for the next working day.

If today is **Friday**, "tomorrow" = next Monday. If today is **Saturday/Sunday**, ask Alex which day they want to plan for. Otherwise tomorrow = today + 1 day.

### Step 5 — Write today's daily

Update `Daily/YYYY-MM-DD.md` (today's file) with the gathered content:

- Handle completed work in two distinct ways depending on its origin:
  1. **Items that were in `## Schedule`** — flip the checkbox **in place** from `- [ ]` to `- [x]`. Do **not** move the line out of `## Schedule`, do **not** strike it through (`~~...~~` breaks Obsidian rendering). The morning plan must remain visible as memory. If Alex completed only part of a Schedule line (e.g. half a long block), split it into a checked part and a remaining `- [ ]` part — both stay in `## Schedule`.
  2. **Ad-hoc work that was NOT in `## Schedule`** — write it into `## Done today` as a plain checked checkbox, no strikethrough:
     ```markdown
     - [x] Fix bug X (#project/cloud-firewall)
     - [x] Review PR ermes (#project/ermes)
     ```
  Never use `~~...~~` strikethrough anywhere — it breaks Obsidian rendering. Use plain `- [x]` only.
- Fill `## Blocked` with bullets, each containing a `#blocked` tag and the unblocking action.
- Fill `## Retro` (skip if empty).
- Verify `## Work log` entries are well-formed — each bullet should be prefixed by its local time `HH:MM`, in the canonical form `- **HH:MM — <title>**: <body>`. Do **not** rewrite Alex's prose, but if a Work log bullet is missing the time prefix, ask Alex what time it was and add it. Never invent a time.
- Update frontmatter: `updated: <today>`, ensure `tags` includes `#daily` and `#teardown`.

Preserve any free-form `## Work log` content Alex wrote during the day — never delete it.

### Step 6 — Plan the next working day

Compute the target date (see Step 4). Path: `Daily/<target_date>.md`.

If the target file does not exist, create it from the template (frontmatter + section headers, no content). If it already has a `## Planned tasks` section, **append** to it; do not overwrite.

Write each planned task as a **plain bullet, without checkbox**:

```markdown
- <Task description> (<#project/<name>> if relevant)
```

> **IMPORTANT**: do **not** use `- [ ]` here. Planned tasks acquire `[ ]` only the morning after, when `daily-standup` promotes them into `## Schedule`. This is the discriminator between "plan" and "today's schedule".

Add `#planned` to the file's frontmatter `tags` if not already present.

### Step 7 — Confirm before rollup

Before running the rollup, summarise to Alex what you wrote and where, and ask for confirmation:

> "Ho aggiornato `Daily/<today>.md` con [...] e ho pianificato N task in `Daily/<target>.md`. Procedo con il rollup nell'archivio?"

Wait for an explicit "sì" / "ok" / "vai".

### Step 8 — Invoke rollup

Once confirmed, invoke the `rollup` skill with the following intent:

- The daily note for `<today>` should move from `Daily/` to its Weekly folder.
- The `rollup` skill will detect end-of-week / end-of-month / end-of-year and cascade further if needed.

Do **not** try to do the rollup yourself — defer to the dedicated skill so the cascade logic stays in one place.

### Step 9 — Final report

After rollup completes, give Alex a one-paragraph wrap-up: where today's note ended up, how many tasks are planned for the target day, and any cascade events that happened (e.g., "fine settimana rilevata, archiviato anche in Monthly/05/").

## Style rules

- Italian by default. Technical terms can stay in English.
- The free-form opening (Step 3) must be a real conversation starter, not a checklist.
- Use AskUserQuestion only for genuinely multi-choice prompts (e.g., picking the target day on a weekend). Avoid making the interview feel mechanical.
- Never invent content. If Alex says nothing about retro, leave the section empty.
- Preserve all pre-existing content in today's daily — only fill in empty sections.
