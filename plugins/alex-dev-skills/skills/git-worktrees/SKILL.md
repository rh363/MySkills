---
name: git-worktrees
description: Usa questa skill quando devi lavorare in parallelo su più branch senza continui `git stash`/`checkout` — code review mentre sviluppi, hotfix urgente durante feature, confronto di due implementazioni, CI locale su branch diverso. Enforce setup e pulizia con `git worktree` invece di cloni multipli.
---

# Git Worktrees

## Principio

**Un checkout non basta.** Quando lavori su più branch in parallelo, lo switching continuo con `git stash`/`checkout` genera errori (stash dimenticati, build invalidate, IDE confusa). `git worktree` ti dà **più cartelle di lavoro dallo stesso repo**, ciascuna sul suo branch, senza clonare.

Casi d'uso tipici:
- Feature in corso + review urgente di PR di un collega
- Hotfix produzione senza mollare la feature
- Compare A/B di due implementazioni senza rebuild
- CI locale / test di lungo corso in parallelo

## Processo

### 1. Scegliere una directory per i worktree
Convenzione: `../<repo>-worktrees/<branch-name>` — accanto al clone principale, facile da trovare.

### 2. Creare un worktree
```bash
# Da dentro il clone principale
git worktree add ../myrepo-worktrees/hotfix-login -b hotfix/login

# oppure su un branch esistente (remoto)
git worktree add ../myrepo-worktrees/pr-1234 origin/feature/pr-1234
```

### 3. Lavorare nel worktree
```bash
cd ../myrepo-worktrees/hotfix-login
# IDE aperta qui in parallelo al worktree principale
# commit, push, PR normalmente
```

### 4. Elencare e pulire
```bash
git worktree list
# /home/alex/code/myrepo              abc1234 [main]
# /home/alex/code/myrepo-worktrees/x  def5678 [hotfix/login]

# Dopo merge:
git worktree remove ../myrepo-worktrees/hotfix-login
# e sul repo principale:
git branch -d hotfix/login
```

### 5. Prune dei worktree orfani
Se rimuovi la cartella a mano (cancellazione, disk full, altro):
```bash
git worktree prune
```

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "Uso stash, è più veloce" | Stash accumulati → si perdono. Worktree = stato persistente, ispezionabile. |
| "Clono il repo due volte" | Occupa 2x spazio (specie monorepo), duplica config Git, separa storia oggetti. Worktree condivide `.git`. |
| "L'IDE non gestisce due worktree" | IDE moderni sì (VS Code multi-root, JetBrains project switch). Tab separati. |
| "È solo per casi estremi" | Se lavori in team attivo con PR frequenti, succede **ogni settimana**. |

## Esempio 1 — Hotfix durante feature

```bash
# Sei su feature/big-refactor con modifiche non committate
cd ~/code/ecs2  # main worktree
git stash -u  # opzionale, ma meglio NON stashare

# Apri worktree per hotfix (parte da main)
git worktree add ../ecs2-worktrees/hotfix-celery-oom -b hotfix/celery-oom main

# Lavora in parallelo senza toccare la feature
cd ../ecs2-worktrees/hotfix-celery-oom
# ... fix, test, PR, merge ...

# Torna alla feature
cd ~/code/ecs2
# la feature è intatta, nessun stash da gestire

# Pulisci il worktree del hotfix
git worktree remove ../ecs2-worktrees/hotfix-celery-oom
```

### Esempio 2 — Review PR senza interrompere il tuo lavoro

```bash
# PR #1234 da un collega su feature/search-v2
git fetch origin
git worktree add ../myrepo-worktrees/review-1234 origin/feature/search-v2

cd ../myrepo-worktrees/review-1234
# installa deps se necessario (per Python/TS ogni worktree ha il suo venv/node_modules)
python -m venv .venv && source .venv/bin/activate && pip install -r requirements.txt
pytest  # gira i test del PR mentre tu lavori sul tuo branch nel main worktree

# Dopo review:
cd ~/code/myrepo
git worktree remove ../myrepo-worktrees/review-1234
```

### Esempio 3 — Compare A/B tra due implementazioni

```bash
# Sei su feature/impl-a, vuoi vedere come si comporta impl-b su stesso test set
git worktree add ../myrepo-worktrees/impl-b feature/impl-b

# Terminal 1: impl-a benchmark
cd ~/code/myrepo && go test -bench=. ./internal/core > /tmp/bench-a.txt

# Terminal 2: impl-b benchmark
cd ../myrepo-worktrees/impl-b && go test -bench=. ./internal/core > /tmp/bench-b.txt

# Confronto
benchstat /tmp/bench-a.txt /tmp/bench-b.txt
```

## Integrazione IDE

- **VS Code**: `File → Open Folder` sul worktree, o "Add Folder to Workspace" per multi-root.
- **JetBrains**: `File → Open` nuova window, selezione del worktree.
- **Neovim/Vim**: nuova istanza nella cartella del worktree, o `:cd` se usi una sessione.

## Pitfall ricorrenti

- **Non committato nel worktree, cartella cancellata a mano** → `git worktree prune` ripulisce la registrazione.
- **Stesso branch in due worktree** → Git rifiuta: `fatal: 'hotfix/x' is already checked out at ...`. Un branch per worktree.
- **Submodule / hooks** → attenzione: hooks sono condivisi (`.git/hooks`), submodule potrebbero richiedere `git submodule update` in ogni worktree.
- **venv/node_modules**: ciascun worktree ha la sua copia. Lento da setuppare → tool come `uv`/`pnpm` riducono il dolore.
- **`.env` locale**: non condivisa tra worktree (non è tracciata). Copia manuale o symlink se serve.

## Quando NON usare worktree

- Lavoro su **un solo branch** e switching è raro
- Spazio disco molto limitato e il progetto ha dipendenze enormi replicate
- Repo molto piccoli dove `git stash` + `checkout` è più snello

## Exit criteria

- [ ] Il worktree è creato in una cartella **fuori dal repo principale**
- [ ] Il branch del worktree è diverso da quello del main worktree
- [ ] Post-merge: `git worktree remove` + `git branch -d` per pulizia
- [ ] Orfani rimossi con `git worktree prune` periodicamente
- [ ] IDE aperta nella cartella del worktree, non in sottocartella del principale
