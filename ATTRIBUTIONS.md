# Attributions

Le skill di `alex-skills` sono **originali** — non copie letterali di altri repo. Format, tono, esempi e focus sono stati riscritti da zero per lo stack polyglot personale dell'autore (Python/Go/TypeScript + pattern Seeweb).

Tuttavia, i **concept e il pattern generale** di alcune skill sono ispirati a lavori esistenti di qualità, a cui va il credito per aver stabilito lo stato dell'arte.

## Fonti di ispirazione

### [obra/superpowers](https://github.com/obra/superpowers) — MIT
Ispirazione per:
- `tdd-prove-it` — pattern RED → GREEN → REFACTOR, approccio "no fix without failing test first", anti-rationalization table
- `systematic-debugging` — processo in 5 step (reproduce → localize → reduce → fix → guard)
- `git-worktrees` — uso di `git worktree` per workflow paralleli

Le skill in questo repo sono **scritte da zero**, con esempi ed enfasi adattati al mio dominio. Non è copia del contenuto di `superpowers`.

### [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills) — MIT
Ispirazione per:
- `context-engineering` — approccio a `CLAUDE.md`, packing strategy, selective inclusion

Il resto del bundle `addyosmani/agent-skills` (frontend engineering React/Vue, spec-driven development, ecc.) non è stato incluso perché eccessivamente TS/web-centric per il mio stack.

### [anthropics/skills](https://github.com/anthropics/skills) — Varie licenze
Riferimento per il **formato canonico** di una skill Claude Code (frontmatter YAML, struttura markdown, limite token). Nessun contenuto copiato.

### [varlock-claude-skill](https://github.com/varlock) (concept pubblici)
Ispirazione per:
- `secrets-hygiene` — approccio separazione environment/code, no-leak in log/git. Versione più snella, focalizzata sui provider che uso (Stripe, Supabase, Keycloak, SMTP, RevenueCat).

## Skill 100% originali (custom Seeweb)

Le seguenti skill non hanno una fonte diretta di ispirazione esterna — sono state estratte da pattern ricorrenti nel mio lavoro:

- `hexagonal-architecture` — ports & adapters con esempi Python/Go dal mio dominio
- `celery-idempotency` — idempotency per task Celery con pattern Django + PostgreSQL specifici
- `keycloak-oidc-patterns` — integrazione Keycloak/OIDC per progetti interni

## Licenze

- Questo repo è MIT (vedi [LICENSE](LICENSE))
- I contenuti ispiratori sono MIT o licenze compatibili, e il lavoro derivato è lecito per riscrittura + attribuzione
- Nessuna parte di altri repo è inclusa verbatim

Se sei autore di uno dei repo citati e vuoi che rimuova un'attribuzione o la corregga, apri una issue.
