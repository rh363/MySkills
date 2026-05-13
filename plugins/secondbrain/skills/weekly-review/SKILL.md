---
name: weekly-review
description: >
  This skill should be used at the end of a working week (Friday evening,
  or first thing Monday for the week just past) to synthesize all the
  daily notes of a single week into a `Weekly/MM-DD/README.md` summary
  for Alex's Obsidian second brain. Trigger phrases include "weekly
  review", "review settimanale", "fai la weekly", "chiudo la settimana",
  "sintesi della settimana", "wrap-up settimanale". The skill is also
  invoked automatically by `rollup` when it detects end-of-week, before
  cascading the `Weekly/MM-DD/` folder into `Monthly/MM/`. Always load
  `vault-structure` alongside this skill — it owns the layout and the
  `week_folder(date)` algorithm.
metadata:
  version: "0.1.0"
---

# Weekly Review

End-of-week synthesis. Reads all the daily notes belonging to a single working week and produces a structured `README.md` inside the corresponding `Weekly/MM-DD/` folder. The README travels with the dailies through the rollup cascade (→ Monthly → Yearly), so future-Alex can read a fast summary of any past week.

## Required context

`vault-structure` defines paths, the `week_folder(date)` algorithm, and the section conventions of a daily note (`## Schedule`, `## Work log`, `## Done today`, `## Blocked`, `## Retro`). Use it; do not redefine.

## Inputs

- Target week. Two invocation modes:
  - **Manual**: Alex types "weekly review" or similar. Default target = the current week (Monday → Friday of today's week). If today is Saturday/Sunday, default = the week that just ended.
  - **From rollup**: `rollup` passes the exact `Weekly/<MM-DD>/` folder path. Use that as the target — do **not** recompute.
- Vault root: `~/Documents/Obsidian Vault/`.

## Workflow

Follow the steps in order. Do not skip even if you think you already know the answer.

### Step 1 — Resolve the target week

Given the invocation mode:

- **From rollup**: the input is a folder path like `Weekly/04-08/`. Read it directly.
- **Manual**: compute today's `week_folder` using the algorithm in `vault-structure`. Then look for the dailies in two locations (a week can be partially rolled up):
  - `Daily/YYYY-MM-DD.md` — for dailies not yet rolled up.
  - `Weekly/<week_folder>/YYYY-MM-DD.md` — for dailies already rolled up.

The set of dailies of the target week is the union of these two sources. If both sources are empty, tell Alex: "Nessun daily trovato per la settimana `<MM-DD>`. Sicuro che vuoi fare la review?" and stop unless they confirm.

If Alex says "weekly review della settimana scorsa" or similar, target the previous week — recompute `week_folder` with `today - 7 days`.

### Step 2 — Read every daily of the week

For each daily file found in Step 1, Read it in full. Parse out the standard sections:

- `## Schedule` — note which checkboxes are `[x]` vs `[ ]` (open carry-over candidates).
- `## Work log` — free-form prose.
- `## Done today` — consolidated done list.
- `## Blocked` — open blockers.
- `## Retro` — insights.

Also collect frontmatter tags and `#project/...` mentions so you can group the week by project.

### Step 3 — Detect destination folder

The README target is `Weekly/<week_folder>/README.md`. Ensure the folder exists (`mkdir -p`). If `README.md` already exists, **do not overwrite it silently** — read it first, and treat its content as the previous draft. Tell Alex: "Esiste già una review per `<week_folder>`. Vuoi che la riscriva da zero, la integri, o la lasci stare?" Default to integration if Alex doesn't have a strong preference.

### Step 4 — Free-form opening

In Italian, summarise in 2–3 sentences what you observed across the week (no list, plain prose). Then ask one open question:

> "Ho letto i daily di lunedì–venerdì. La settimana sembra essere stata su X, Y, Z. Vuoi raccontarmi come l'hai vissuta, o partiamo dai punti che mi sembrano più importanti?"

Listen fully before drilling down.

### Step 5 — Targeted gap-filling

After the free-form opening, ask 2–4 mirror questions only on the gaps you still have. Use `AskUserQuestion` only for genuinely multi-choice prompts. Cover at minimum:

- **Highlights** — 3–5 cose che vale la pena ricordare di questa settimana.
- **Carry-over** — task non completate dal `## Schedule` di qualche giorno. Alex decide se ri-schedularle lunedì prossimo, droppale, o marcarle `#postponed`.
- **Temi ricorrenti / pattern** — qualcosa è apparso più volte? (es. stesso blocker 3 giorni, stesso progetto in stallo, stesso tipo di task che ha sempre sforato).
- **Priorità prossima settimana** — 1–3 cose con cui aprire lunedì.

Skip retro/insights se il giorno-per-giorno era già denso di Retro — basta consolidarle.

### Step 6 — Write `Weekly/<week_folder>/README.md`

Compose the README with this structure (Italian content, sections in inglese per coerenza col resto del vault):

```markdown
---
created: YYYY-MM-DD     # the friday of the week
updated: YYYY-MM-DD     # today
tags: [weekly, review]
week: MM-DD             # the week folder name
year: YYYY              # the year of the Monday
---

# Settimana MM-DD (YYYY)

> Lun YYYY-MM-DD → Ven YYYY-MM-DD

## Highlights
- ...
- ...

## Done this week
<!-- consolidato dei ## Done today di ogni daily, deduplicato. Plain [x] checkboxes, NO strikethrough. -->
- [x] ...
- [x] ...

## By project
<!-- raggruppa Done + Work log per #project/<nome>. Una sottosezione per progetto attivo. Salta i progetti con < 2 menzioni nella settimana. -->
### #project/<nome>
- bullet sintetico di cosa è successo, decisioni, ADR scritti.

## Carry-over
<!-- task ancora aperte (- [ ]) dal Schedule di qualche giorno. Indica il giorno di origine tra parentesi. -->
- [ ] ... (da 2026-05-06)

## Blocked
<!-- consolidato dei ## Blocked, deduplicato. Mantieni tag #blocked e la causa. -->
- ...

## Retro
- pattern / insight che vale la pena ricordare.

## Next week
<!-- 1-3 priorità per la settimana che inizia lunedì prossimo. -->
- ...
```

Regole di scrittura:

- **Mai usare `~~...~~`** strikethrough — rompe il rendering Obsidian. Solo `- [x]` / `- [ ]` puliti.
- Linka i daily con wikilink Obsidian dove ha senso: `[[2026-05-08]]`.
- Linka i progetti come `[[Projects/<nome>/README]]` se ne parli più di una volta.
- Se una sezione è vuota (`## Retro` senza spunti), lasciala con un singolo bullet `- _Niente da segnalare_` invece di rimuoverla. Coerenza è importante per la cascata.

### Step 7 — Aggiorna le carry-over nel daily di lunedì prossimo

Se in Step 5 sono emerse task da portare alla settimana prossima:

1. Calcola la data del **lunedì prossimo** (oggi + giorni necessari ad arrivare a lunedì).
2. Path: `Daily/<lunedì_prossimo>.md`. Se non esiste, crealo dal template (vedi `vault-structure`).
3. Appendi le carry-over alla sezione `## Planned tasks` come **bullet semplici senza checkbox** (acquisiranno `[ ]` solo quando `daily-standup` le promuoverà in `## Schedule` lunedì mattina).
4. Aggiungi tag `#carry-over` al frontmatter del daily di lunedì se non presente.

Non toccare il `## Planned tasks` di lunedì se esiste già — appendi soltanto. Non duplicare task già presenti (confronto case-insensitive sul testo).

### Step 8 — Conferma e report finale

Riepiloga in chat:

> "Ho scritto `Weekly/<MM-DD>/README.md` con N highlight, M done items, K carry-over (riportate in `Daily/<lunedì_prossimo>.md`). Vuoi che proceda con il rollup, o ti fermi qui?"

Se sei stato invocato da `rollup`, **non chiedere conferma per il rollup** — restituisci semplicemente il controllo a rollup, che proseguirà con la cascata Weekly → Monthly. In quel caso il report finale può essere una singola riga: "Weekly review scritta in `<week_folder>/README.md`."

## Style rules

- Italiano di default. Tecnicismi in inglese ok.
- L'apertura free-form (Step 4) deve essere conversazionale, non un riepilogo da bullet point.
- Nessuna content invention: se Alex non parla di Retro, lascia `_Niente da segnalare_`. Se non ci sono carry-over, ometti del tutto la sezione e non aggiungere niente al lunedì.
- Una review deve poter essere scritta in 10–15 minuti. Se ti accorgi che stai facendo 15 domande, ferma — meglio una review buona-abbastanza che una perfetta che Alex non finisce.
- Non toccare i singoli daily — sono storia, non li riscrivere. Solo lettura.

## Edge cases

- **Settimana di 2-3 giorni** (vacanze, festività): scrivi comunque la README, ma rendila proporzionale — niente sezioni gonfiate per sembrare piena. Se non c'è materiale per una sezione, salta la sezione (eccetto Highlights e Done this week che restano sempre).
- **Settimana che straddla due mesi** (es. lun 27/04 → ven 01/05): la README sta nel `Weekly/<MM-DD>/` calcolato sul lunedì (vedi `vault-structure`). Quando `rollup` cascaderà la cartella, finirà sotto il mese del lunedì.
- **README già esistente**: vedi Step 3.
- **Invocazione fuori dall'orizzonte settimanale** (es. Alex chiede "weekly review" un mercoledì): chiedi esplicitamente se vuole un mid-week snapshot (review parziale, niente carry-over al lunedì prossimo) o se intende la settimana scorsa.
