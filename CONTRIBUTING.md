# Contributing

Questo plugin è pensato come **toolset personale**. Contributi esterni sono ben accetti se allineati al principio cardine: *una skill esiste solo se viene usata almeno una volta a settimana*.

## Prima di aprire un PR

- **Apri una issue** per discutere la skill. Evita di scrivere contenuto che potrebbe essere rifiutato per scope.
- Verifica che non ci sia già una skill simile.
- Confronta con il backlog nel README del progetto (sezione "Backlog skill future" in Obsidian) per capire cosa è in roadmap.

## Formato di una skill

Ogni skill è una cartella dentro `plugins/alex-dev-skills/skills/<kebab-case-name>/` con un file `SKILL.md`:

```markdown
---
name: kebab-case-name
description: Quando usare questa skill, in modo chiaro e actionable. Include i segnali che devono far scattare l'attivazione (es. "usa questa skill quando...").
---

# Nome Skill

## Principio
1-3 righe con la regola cardine. No fuffa.

## Processo
Step numerati, verificabili. Ogni step ha un'azione concreta.

## Anti-rationalization (opzionale ma utile)
Tabella di scuse comuni + rebuttals. Molto efficace.

## Esempi pratici
Almeno un esempio **concreto** in Python / Go / TypeScript. Codice funzionante, non pseudocodice.

## Exit criteria
Checklist finale di cosa deve essere vero quando la skill è stata applicata.
```

## Vincoli di stile

- **Tono diretto**, no prolissità. Se una frase può essere tagliata senza perdere significato, va tagliata.
- **Lingua**: italiano nel body, inglese per termini tecnici consolidati (RED/GREEN, ports/adapters, JWT, ecc.).
- **Esempi concreti**: codice reale che compila, non pseudo-codice. Se l'esempio è lungo, splittalo in blocchi per fase.
- **Limite token**: ogni `SKILL.md` deve stare **sotto 5000 token** (limite Claude Code per skill attivata).
  - Se superi → **splitta** la skill in due focalizzate. Non troncare.
- **Anti-rationalization**: tabella a 2 colonne (Scusa | Realtà) è il formato che funziona meglio.

## Cosa NON includere

- **Script eseguibili** nelle skill (riduce attack surface, evita rischi supply chain).
- **Secret/credenziali** anche negli esempi. Usa placeholder.
- **Copie verbatim** di altri repo. Le fonti di ispirazione sono in `ATTRIBUTIONS.md`, i contenuti sono riscritti.
- **Esempi specifici di cliente/prodotto** in modo non anonimizzato.

## Validazione

Prima di aprire il PR:

```bash
claude plugin validate .
```

Fix di tutti i warning.

## Test

Installa il plugin in locale e usa la skill su un caso reale per almeno una settimana. Se non scatta mai da sola → la `description` è debole, raffinala.

## Commit

Usa conventional commits:
- `feat: add <skill-name> skill` per nuove skill
- `fix: correct example in <skill-name>` per fix
- `docs: improve <skill-name> description` per miglioramenti description
- `chore: update .gitignore` per chore
