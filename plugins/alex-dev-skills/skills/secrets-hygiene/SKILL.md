---
name: secrets-hygiene
description: Usa questa skill quando gestisci credenziali, chiavi API, token, certificati — setup nuovi servizi (Stripe, Supabase, Keycloak, SMTP), rotazioni, configurazione CI/CD, o quando noti un secret in posto sbagliato (log, commit, screenshot, sessione Claude). Enforce separazione environment/code, no-leak in log/git/sessioni, rotazione ordinata.
---

# Secrets Hygiene

## Principio

**Un secret è un secret solo finché non lo è più.** Un singolo leak (commit pubblicato, log catturato, screenshot condiviso) lo rende compromesso per sempre. Rotazione periodica non basta: serve **evitare il leak in prima istanza**.

Regole cardine:
1. **Mai** secret in codice, commit, o documentazione
2. **Mai** secret in log (anche warning/error message)
3. **Mai** secret in sessioni Claude Code o screenshot condivisi
4. **Mai** stesso secret in più ambienti (dev == prod è un bug)
5. **Sempre** rotazione pianificata e procedura di incident

## Processo

### 1. Classifica i tipi di secret
| Tipo | Esempi | Rotation target |
|---|---|---|
| **API key** (third party) | Stripe, OpenAI, Anthropic, AWS, SendGrid | 90-180 gg o su sospetto |
| **DB credentials** | PostgreSQL, Redis auth | 90 gg + su turnover team |
| **JWT signing key** | Keycloak realm keys | Rotation pianificata (Keycloak gestisce overlap) |
| **OIDC client secret** | Keycloak client confidential | 180 gg |
| **TLS cert** | Let's Encrypt, custom CA | 90 gg (automatico Let's Encrypt) |
| **SSH key** | Deploy key, user key | Annuale o su turnover |
| **Webhook secret** | Stripe webhook, GitHub webhook | Su sospetto; generare unique per env |
| **Session cookie secret** | Django SECRET_KEY, FastAPI session | Su sospetto (invalida tutte le sessioni) |

### 2. Storage corretto
- **Development**: `.env` file **mai committato**, in `.gitignore`. Template come `.env.example` con placeholder.
- **Staging/Production**: secret manager (Vault, AWS Secrets Manager, Kubernetes Secrets + sealed-secrets, Docker secrets, GitLab CI variable masked).
- **CI/CD**: variabili masked, mai in echo/print. Mascheramento automatico se injectate come masked.
- **Mai**: hardcoded, config file in repo, Slack, email, chat, issue tracker.

### 3. Prevenzione leak in codice
- **Pre-commit hook**: `gitleaks`, `trufflehog`, `detect-secrets`. Obbligatorio.
- **CI check**: stesso tool in pipeline, bloccante.
- **Code review**: occhio a variabili con nomi `key`, `token`, `password`, `secret`, `api_*`.
- **Template file**: mai valori reali, placeholder chiari (`REPLACE_ME`, `<your-stripe-key>`).

### 4. Prevenzione leak in log
- **Logger custom** che filtra campi sensibili (`authorization`, `cookie`, `set-cookie`, `password`, `token`, `secret`, `api_key`)
- **Request/response logger**: redact header `Authorization` e body fields noti
- **Mai** `logger.info(f"user logged in with token {token}")`
- **Mai** loggare intero body di payload di pagamento/auth

### 5. Prevenzione leak in sessioni Claude Code
- **Mai** incollare `.env` in una sessione
- **Mai** incollare output che contiene `Authorization:` header pieno
- Se devi far vedere una request a Claude: **redact manualmente** (`Bearer xxxxx`)
- Rivedi screenshot prima di condividerli (le toolbar dev leakano header)

### 6. Rotazione ordinata
Procedura standard:
1. Genera nuovo secret nel provider (Keycloak, Stripe, ecc.)
2. Deploy applicazione con **entrambi** supportati (dove possibile)
3. Verifica traffic sui nuovi (metrica/log)
4. Revoca il vecchio
5. Monitora errori 401 per un ciclo di release

### 7. Incident (secret leakato)
1. **Revoca subito** il secret nel provider
2. Genera nuovo e deploya
3. Audit log per uso fraudolento del vecchio
4. Se committato: `git filter-repo` o BFG (non basta revert — la history resta)
5. Documenta nel post-mortem senza includere il secret (ovviamente)
6. Review del processo: come è successo, come prevenire

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "È solo un token di dev" | I token di dev finiscono nei log, negli screenshot, nei repo pubblici. Stesso rigore. |
| "Il repo è privato, tranquillo" | Turnover team, contractor, backup, fork accidentali. Privato ≠ segreto. |
| "Lo metto nel readme come esempio" | Qualcuno lo userà. Placeholder sempre. |
| "Il log è solo locale" | Finisce in Sentry, in grep condivisi, in screenshot. Redact sempre. |
| "Ruoto se qualcosa va storto" | Quando "va storto", spesso non lo sai. Rotazione pianificata è l'unica prevenzione. |

## Esempio 1 — `.env` / `.env.example` pattern

```bash
# .gitignore
.env
.env.local
.env.*.local
!.env.example

# .env.example (committato)
STRIPE_SECRET_KEY=sk_test_REPLACE_ME
DATABASE_URL=postgresql://user:pass@localhost/db
KEYCLOAK_CLIENT_SECRET=REPLACE_ME
DJANGO_SECRET_KEY=generate_with_openssl_rand_base64_64
```

## Esempio 2 — pre-commit hook con gitleaks

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.0
    hooks:
      - id: gitleaks
```

```bash
pre-commit install
# prova: echo 'STRIPE_KEY=sk_live_1234' > leak.py && git add leak.py && git commit
# → commit bloccato
```

## Esempio 3 — logger redacting Python

```python
import logging, re

SENSITIVE_PATTERNS = [
    (re.compile(r'(Bearer\s+)[A-Za-z0-9._\-]+', re.I), r'\1<REDACTED>'),
    (re.compile(r'("?(?:password|token|secret|api[_-]?key)"?\s*[:=]\s*"?)[^"\s,}]+', re.I), r'\1<REDACTED>'),
]

class RedactingFilter(logging.Filter):
    def filter(self, record):
        msg = record.getMessage()
        for pat, repl in SENSITIVE_PATTERNS:
            msg = pat.sub(repl, msg)
        record.msg = msg
        record.args = ()
        return True

logging.getLogger().addFilter(RedactingFilter())
# ora: logger.info("auth header: Bearer abcdefg...") → "auth header: Bearer <REDACTED>"
```

## Esempio 4 — Go: redact header in middleware

```go
func sanitizedHeaders(h http.Header) http.Header {
    out := h.Clone()
    for _, k := range []string{"Authorization", "Cookie", "X-Api-Key", "Proxy-Authorization"} {
        if out.Get(k) != "" {
            out.Set(k, "<REDACTED>")
        }
    }
    return out
}

func LoggingMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        log.Printf("req %s %s headers=%v", r.Method, r.URL.Path, sanitizedHeaders(r.Header))
        next.ServeHTTP(w, r)
    })
}
```

## Esempio 5 — provider-specific patterns

- **Stripe**: usa **restricted keys** quando possibile (scope limitato). Webhook secret diverso per endpoint.
- **Keycloak**: client confidential → client secret in secret manager. Realm signing keys gestite da Keycloak con rotation config.
- **Supabase**: `service_role` key **mai** client-side. Solo backend.
- **SMTP**: credenziali dedicate per invio transazionale, dominio SPF/DKIM configurati.
- **RevenueCat**: secret API key solo server-side, app key (public) in app.

## Checklist per ambiente nuovo

- [ ] `.env.example` committato, `.env` in `.gitignore`
- [ ] Pre-commit con `gitleaks` configurato
- [ ] CI fail se pattern di secret rilevato
- [ ] Secret manager scelto per staging/prod (e non è "file .env sul server")
- [ ] Logger con redacting filter attivo
- [ ] Rotation policy scritta (anche breve, in `SECURITY.md` o runbook)
- [ ] Incident runbook per "secret leakato": chi revoca, chi deploya, chi audita

## Exit criteria

- [ ] Nessun secret nel repo (storia inclusa: verifica con `gitleaks detect`)
- [ ] Nessun secret nei log (test con request contenente `Authorization: Bearer ...`)
- [ ] `.gitignore` blocca `.env` e varianti
- [ ] CI ha un check secret-scanning bloccante
- [ ] I secret di prod/stage/dev sono **diversi**
- [ ] Esiste una procedura di rotazione (anche manuale) documentata
