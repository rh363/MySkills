# MySkills

Plugin **Claude Code** personale, installabile via marketplace, con 11 skill + 2 agent di sviluppo adattati a stack polyglot (Python/Go/TypeScript) e pattern backend distribuito (Celery, Keycloak/OIDC, hexagonal architecture).

- **Autore**: Alex Massaroni ([rh363](https://github.com/rh363))
- **Licenza**: MIT
- **Versione**: 0.1.0

## Perché esiste

I bundle di skill generici (1400+ skill) sono rumorosi e portano rischi di sicurezza. I repo full-stack TS-centric non coprono il mio stack (Python/Go backend + SvelteKit/Flutter frontend). `MySkills` è il compromesso: **11 skill + 2 agent**, ognuna pensata per essere usata almeno una volta a settimana, riscritte da zero ispirandosi a fonti battle-tested ([obra/superpowers](https://github.com/obra/superpowers), [addyosmani/agent-skills](https://github.com/addyosmani/agent-skills), [anthropics/skills](https://github.com/anthropics/skills)).

Vedi [ATTRIBUTIONS.md](ATTRIBUTIONS.md) per le attribuzioni di ispirazione.

## Installazione

Da dentro Claude Code:

```
/plugin marketplace add rh363/MySkills
/plugin install alex-dev-skills@MySkills
```

Claude Code attiverà automaticamente le skill rilevanti in base a quello che stai facendo. Puoi anche invocarle esplicitamente: `usa tdd-prove-it per questo fix`.

## Le 11 skill

### Core — universali

| Skill | Quando scatta |
|---|---|
| [`tdd-prove-it`](plugins/alex-dev-skills/skills/tdd-prove-it/SKILL.md) | Quando stai per fixare un bug o implementare una feature: scrivi prima il test che fallisce (RED → GREEN → REFACTOR). |
| [`systematic-debugging`](plugins/alex-dev-skills/skills/systematic-debugging/SKILL.md) | Bug non banali, flaky, distribuiti: processo in 5 step (reproduce → localize → reduce → fix → guard). |
| [`context-engineering`](plugins/alex-dev-skills/skills/context-engineering/SKILL.md) | Setup `CLAUDE.md`, packing strategy, evitare context overload nelle sessioni. |
| [`git-worktrees`](plugins/alex-dev-skills/skills/git-worktrees/SKILL.md) | Lavoro parallelo su più branch (feature + hotfix, review + dev). |

### Security

| Skill | Quando scatta |
|---|---|
| [`owasp-security-focused`](plugins/alex-dev-skills/skills/owasp-security-focused/SKILL.md) | Input utente, auth, DB, upload, deserializzazione: threat model + OWASP Top 10 2025 + quirk Python/Go. |
| [`secrets-hygiene`](plugins/alex-dev-skills/skills/secrets-hygiene/SKILL.md) | Gestione credenziali e token: no-leak in log/git/sessioni, rotazione ordinata. |

### Platform & delivery

| Skill | Quando scatta |
|---|---|
| [`api-contract-first`](plugins/alex-dev-skills/skills/api-contract-first/SKILL.md) | Progetti o modifichi API HTTP/gRPC con client e server in linguaggi diversi: OpenAPI/Protobuf come single source of truth, codegen, `oasdiff` in CI. |
| [`dependency-upgrade-discipline`](plugins/alex-dev-skills/skills/dependency-upgrade-discipline/SKILL.md) | Configuri Renovate/Dependabot o valuti upgrade: batched settimanali, changelog review, SCA (pip-audit / govulncheck / npm audit) in CI. |

### Custom Seeweb — il differenziatore

| Skill | Quando scatta |
|---|---|
| [`hexagonal-architecture`](plugins/alex-dev-skills/skills/hexagonal-architecture/SKILL.md) | Nuovo modulo/servizio o refactor: ports & adapters con esempi Python/Go. |
| [`celery-idempotency`](plugins/alex-dev-skills/skills/celery-idempotency/SKILL.md) | Task Celery con side effect: idempotency key, state guard SQL, distributed lock. |
| [`keycloak-oidc-patterns`](plugins/alex-dev-skills/skills/keycloak-oidc-patterns/SKILL.md) | Integrazione Keycloak/OIDC: Auth Code + PKCE, JWT RS256, state/nonce, JWKS caching. |

## I 2 agent

Gli agent orchestrano più skill in una pipeline obbligata. Sono invocati esplicitamente (es. `usa tdd-bug-fixer su questo bug`) o auto-attivati quando Claude Code riconosce il contesto dalla `description`.

| Agent | Quando usarlo | Compone |
|---|---|---|
| [`tdd-bug-fixer`](plugins/alex-dev-skills/agents/tdd-bug-fixer.md) | Bug report o fix richiesto: forza pipeline reproduce → failing test → fix minimale → regression guard. | `tdd-prove-it` + `systematic-debugging` |
| [`security-reviewer`](plugins/alex-dev-skills/agents/security-reviewer.md) | Review read-only di diff/PR prima del merge: secrets, input boundary, OWASP, auth/authz, supply chain. | `owasp-security-focused` + `secrets-hygiene` |

## Struttura del repo

```
MySkills/
├── .claude-plugin/
│   └── marketplace.json
├── plugins/
│   └── alex-dev-skills/
│       ├── .claude-plugin/
│       │   └── plugin.json
│       ├── skills/
│       │   └── <11 skills>/SKILL.md
│       └── agents/
│           └── <2 agents>.md
├── README.md
├── ATTRIBUTIONS.md
├── LICENSE
├── CONTRIBUTING.md
└── .gitignore
```

## Contribuire / aggiungere nuove skill

Per progetti personali; contributi esterni non attivamente cercati ma ben accetti via PR se allineati con il principio "una skill vale se la uso almeno una volta a settimana".

Vedi [CONTRIBUTING.md](CONTRIBUTING.md) per il formato di una nuova skill.

## Validazione

```bash
claude plugin validate .
```

## Filosofia

- **Minimalismo**: nessuna skill "nice to have". Se non la uso, la rimuovo.
- **Zero script eseguibili** nelle skill per adesso (ridotta attack surface).
- **Riscrittura ispirata, non copia**: format proprio, esempi adattati allo stack, zero issue licenze.
- **Italiano body, inglese tecnico**: coerente con come penso.
