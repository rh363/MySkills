---
name: rollup
description: >
  This skill should be used to archive a daily note from the Daily/
  folder into the temporal cascade (Weekly → Monthly → Yearly) of Alex's
  Obsidian second brain. It is invoked automatically at the end of the
  daily-teardown skill, but can also be triggered manually with phrases
  like "fai il rollup", "archivia il daily", "rollup settimanale",
  "chiudi la settimana", "archivia il mese". The skill detects
  end-of-week, end-of-month, and end-of-year events and cascades the
  archive accordingly. Always load vault-structure alongside this skill
  for naming and folder rules.
metadata:
  version: "0.1.0"
---

# Rollup

Filesystem operation. Moves daily notes through the temporal archive cascade and bubbles up at period boundaries.

## Required context

`vault-structure` defines paths and the `week_folder(date)` algorithm. Use it; do not guess.

## Cascade rules (recap)

```
Daily/YYYY-MM-DD.md
    → Weekly/MM-DD/YYYY-MM-DD.md          (always, on every rollup)

Weekly/MM-DD/  (entire folder)
    → Monthly/MM/MM-DD/                   (when the week is complete)

Monthly/MM/   (entire folder)
    → Yearly/YYYY/MM/                     (when the month is complete)

Yearly/YYYY/  stays in place forever
```

A "complete" period means: the rollup is being run for the **last working day** of that period, OR the period has fully passed (e.g., on May 8 we're rolling up something from April).

## Inputs

- The daily file(s) to archive. Default: every file in `Daily/` whose date is today or earlier (so manual cleanups also work). When invoked from `daily-teardown`, the input is just today's daily.
- Vault root: `~/Documents/Obsidian Vault/`.

## Workflow

### Step 1 — Enumerate dailies to roll up

```bash
ls -1 "/home/alex/Documents/Obsidian Vault/Daily/" | grep -E '^[0-9]{4}-[0-9]{2}-[0-9]{2}\.md$'
```

For each filename, parse the date. Sort ascending. Skip files dated in the future (defensive).

### Step 2 — For each daily file, move to its weekly folder

For a file `Daily/YYYY-MM-DD.md`:

1. Compute `week_folder` per the algorithm in `vault-structure` (Monday–Friday range, formatted as `MM-DD`).
2. Ensure `Weekly/<week_folder>/` exists (`mkdir -p`).
3. Move the file there. Use Bash `mv`, which preserves mtime.

```bash
mkdir -p "/home/alex/Documents/Obsidian Vault/Weekly/04-08"
mv "/home/alex/Documents/Obsidian Vault/Daily/2026-05-08.md" \
   "/home/alex/Documents/Obsidian Vault/Weekly/04-08/2026-05-08.md"
```

### Step 3 — Detect end-of-week and cascade Weekly → Monthly

For each `Weekly/<week_folder>/` directory after Step 2:

- The week is **complete** if the latest daily in that folder is the Friday of that week (`monday + 4 days`), OR if today's date is past that Friday.
- If complete:
  1. Determine the **month** to file the week under: use the **Monday's month** (zero-padded `MM`).
  2. Ensure `Monthly/<MM>/` exists.
  3. Move the entire week folder: `mv Weekly/<week_folder> Monthly/<MM>/<week_folder>`.

If the week straddles two months (Monday in April, Friday in May), the folder still goes under the Monday's month — this keeps the natural reading order.

### Step 4 — Detect end-of-month and cascade Monthly → Yearly

After all weekly cascades, look at each `Monthly/<MM>/` directory:

- The month is **complete** if today's date is past the last day of that month (calendar-wise), AND the folder contains all weeks whose Monday falls in that month, AND no daily files remain anywhere upstream (Daily/, Weekly/) for that month.
- If complete:
  1. Determine the **year** for that month. The year is the year of the Monday of any contained week (they all match — same month).
  2. Ensure `Yearly/<YYYY>/<MM>/` exists.
  3. Move the entire month: `mv Monthly/<MM> Yearly/<YYYY>/<MM>`.

> A simpler heuristic that is correct in practice: only consider a month complete when there is at least one daily file from a **later** month already archived somewhere. In other words: if everything in `Monthly/05/` is from May 2026, but you've started writing dailies for June 2026, May is done and can cascade.

### Step 5 — Detect end-of-year (rare)

Same logic as end-of-month, one level up. After Step 4, if any `Yearly/<YYYY>/` is fully populated and the current date is in `<YYYY>+1`, the year stays as-is — `Yearly/` is the terminal location.

(End-of-year is a no-op for moves; it exists as a safety check that nothing was missed.)

### Step 6 — Verify and report

After all moves, run a sanity check:

```bash
ls -la "/home/alex/Documents/Obsidian Vault/Daily/"
ls -la "/home/alex/Documents/Obsidian Vault/Weekly/"
```

Report to Alex in plain Italian what happened, e.g.:

- "Archiviato `2026-05-08.md` in `Weekly/04-08/`."
- "Settimana 04-08 completata, cascata in `Monthly/05/04-08/`."
- "Aprile completato, cascata in `Yearly/2026/04/`."

## Edge cases

### Manual rollup mid-week

If the user invokes rollup on a Tuesday, only Step 2 runs (Daily → Weekly). No cascading happens because no period is complete.

### Future-dated planned dailies

`Daily/<future_date>.md` exists because the previous teardown wrote planned tasks there. **Do not roll up future dailies.** Only roll up files dated today or earlier.

### Empty week folders after cascade

If `Weekly/<week_folder>/` becomes empty after a cascade move, remove it with `rmdir` (only if empty — never `rm -rf`).

### Conflicts on `mv`

Before any `mv`, check the destination does not already exist. If a duplicate is detected, abort the move and ask Alex how to resolve it. **Never silently overwrite.**

## Style rules

- Italian for the user-facing report.
- Use Bash for filesystem operations (`mv`, `mkdir -p`, `find`, `rmdir`). Do not use the Write tool to "move" files (that creates duplicates).
- Always print the planned moves before executing them when the cascade is non-trivial (i.e., when more than just the basic Daily → Weekly step is happening). Brief preview, then execute.
- Keep the report concise — one bullet per move.
