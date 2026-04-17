---
name: owasp-security-focused
description: Usa questa skill quando scrivi/modifichi codice che gestisce input utente, autenticazione, autorizzazione, query DB, deserializzazione, upload file, o esponi endpoint. Enforce checklist OWASP Top 10 (2025) + ASVS 5.0 L1 con quirk specifici di Python e Go, focus Keycloak/OIDC e firewall cloud Seeweb.
---

# OWASP Security-Focused

## Principio

**La sicurezza si progetta, non si aggiunge.** Una vulnerabilità OWASP di solito nasce da una scelta di default sbagliata (trust implicito, sanitize al posto giusto mancante, configurazione permissiva). La mitigazione costa poco in fase di scrittura, tantissimo dopo.

Approccio: **threat model leggero → checklist → test**. Non aspettare il pentest per accorgersi.

## Processo

### 1. Threat model leggero (5 min)
Per ogni nuovo endpoint/modulo, rispondi a:
1. **Chi chiama?** (autenticato, anonimo, altro servizio)
2. **Cosa riceve?** (input fidato? limiti?)
3. **Cosa fa?** (read, write, side effect esterno)
4. **Cosa può andare storto se input ostile?** (injection, DoS, auth bypass, leak)
5. **Come lo loggo senza leakare dati?**

### 2. Applica checklist OWASP Top 10 2025

| # | Rischio | Mitigation primaria |
|---|---|---|
| A01 | Broken Access Control | Authz **per ogni endpoint**, deny-by-default, verifica `sub` + ruolo |
| A02 | Cryptographic Failures | TLS 1.2+, hashing password con argon2/bcrypt, no MD5/SHA1 per crypto |
| A03 | Injection | Parametric queries (no string concat), sanitize output HTML, escape shell |
| A04 | Insecure Design | Threat model, rate limit, business logic validata |
| A05 | Security Misconfiguration | Disable debug prod, header security (CSP, HSTS), no stack trace leak |
| A06 | Vulnerable Components | SCA in CI (`pip-audit`, `govulncheck`, `npm audit`), pin versions |
| A07 | Auth Failures | PKCE, MFA dove serve, lockout, token rotation (vedi `keycloak-oidc-patterns`) |
| A08 | Integrity Failures | Firma dei binari/pacchetti, checksum, no deserializzazione untrusted |
| A09 | Logging Failures | Log auth events, **no PII/secret in log** (vedi `secrets-hygiene`) |
| A10 | SSRF | Allowlist di host per fetch esterni, no input diretto in URL |

### 3. Quirk Python
- **`pickle`**: mai deserializzare input esterno. Usa JSON + validazione Pydantic.
- **YAML**: usa `yaml.safe_load`, mai `yaml.load`.
- **`subprocess`**: lista di argomenti, mai `shell=True` con input utente. Usa `shlex.quote` se proprio.
- **SQL Django ORM**: `.extra()` e `.raw()` sono pericolosi, preferisci ORM idiomatico. Se raw: param query con `%s`, mai f-string.
- **`eval`/`exec`**: mai su input esterno, punto.
- **Path traversal**: `pathlib.Path.resolve()` e verifica che sia dentro base dir. `os.path.join` non basta.
- **Requests/urllib**: verifica SSL sempre (default True, non disabilitare).

### 4. Quirk Go
- **`database/sql`**: placeholder `$1, $2` (pgx) o `?` (mysql), mai `fmt.Sprintf`.
- **`html/template`**: autoescape attivo per default. **`text/template`** non escapa — non usarlo per HTML.
- **`encoding/json`**: `json.Unmarshal` su strutture con `DisallowUnknownFields` per evitare mass-assignment.
- **Path traversal**: `filepath.Clean` + verifica prefix. Attenzione a `os.Open` con input user.
- **`exec.Command`**: lista argomenti. Mai concat con user input.
- **SSRF**: usa `net/http` con `Transport` che ha `DialContext` custom che blocca IP privati.
- **`crypto/rand`**, mai `math/rand` per secret/token.

### 5. Keycloak/OIDC (vedi anche skill `keycloak-oidc-patterns`)
- Authz ≠ Authn. Avere un JWT valido **non** significa aver diritto di fare X.
- Controllo `aud` + ruoli + scope su ogni endpoint.
- Per admin endpoint: ruolo dedicato (`admin` realm role), non solo "authenticated".

### 6. Firewall cloud (pattern Seeweb)
- Default deny sia ingress che egress dove sensato
- Rule per ruolo di macchina, non per singolo IP
- Egress verso provider esterni: allowlist con hostname + porta, non `0.0.0.0/0`
- Log delle regole drop per auditing

### 7. Validazione sempre server-side
- I check client-side sono UX, non sicurezza
- Usa schema validation (Pydantic Python, struct tag `validate` Go)
- Valida: tipo, range, lunghezza, pattern, enum values

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "Il frontend valida già" | Il frontend è sotto controllo dell'utente. Valida server-side, sempre. |
| "Il DB ha constraint, basta" | I constraint catturano alcuni casi, non tutti. E danno 500, non validazione pulita. |
| "Siamo dietro firewall interno" | Zero trust. Un servizio compromesso lateralmente ha bisogno di validazione. |
| "Aggiungo sicurezza in un secondo momento" | La security retrofit è 10x più costosa. Default secure ora. |

## Esempio Python — endpoint FastAPI con threat model esplicito

```python
# Endpoint: POST /backends/{id}/restart
# Chi: utente autenticato con ruolo `backend.restart`
# Input: id (UUID), body { reason: str }
# Side effect: comando ad agent su macchina
# Rischi: SSRF (id → url), command injection (reason in comando), authz bypass

class RestartBody(BaseModel):
    reason: str = Field(min_length=3, max_length=200, pattern=r"^[A-Za-z0-9 .,\-]+$")

@router.post("/backends/{backend_id}/restart")
def restart(
    backend_id: UUID,
    body: RestartBody,
    user: User = Depends(require_role("backend.restart")),
    repo: BackendRepo = Depends(get_repo),
):
    backend = repo.get(backend_id)  # 404 se non esiste (evita enumerazione → usa stesso errore di 403)
    if backend.org_id != user.org_id:
        raise HTTPException(404)  # non 403: no info leak sull'esistenza
    # reason validata dal pattern, safe da inserire come parametro
    dispatcher.restart(backend.id, reason=body.reason)
    audit_log("backend.restart", actor=user.id, target=str(backend.id))
    return {"status": "scheduled"}
```

## Esempio Go — mitigation SSRF per fetch esterno

```go
// Solo host allowlist, e blocchiamo IP privati anche su risoluzione DNS

var allowedHosts = map[string]bool{
    "api.stripe.com": true, "accounts.google.com": true,
}

func safeDialContext(ctx context.Context, network, addr string) (net.Conn, error) {
    host, _, _ := net.SplitHostPort(addr)
    ips, err := net.DefaultResolver.LookupIPAddr(ctx, host)
    if err != nil { return nil, err }
    for _, ip := range ips {
        if ip.IP.IsLoopback() || ip.IP.IsPrivate() || ip.IP.IsLinkLocalUnicast() {
            return nil, fmt.Errorf("blocked private IP: %s", ip.IP)
        }
    }
    return (&net.Dialer{Timeout: 5 * time.Second}).DialContext(ctx, network, addr)
}

var safeClient = &http.Client{
    Transport: &http.Transport{DialContext: safeDialContext},
    Timeout:   10 * time.Second,
}

func FetchWebhook(ctx context.Context, rawURL string) (*http.Response, error) {
    u, err := url.Parse(rawURL)
    if err != nil { return nil, err }
    if !allowedHosts[u.Hostname()] {
        return nil, fmt.Errorf("host not allowlisted: %s", u.Hostname())
    }
    req, _ := http.NewRequestWithContext(ctx, "GET", rawURL, nil)
    return safeClient.Do(req)
}
```

## Esempio — SQL injection (cosa evitare, cosa fare)

```python
# ❌ SBAGLIATO
cursor.execute(f"SELECT * FROM users WHERE email = '{email}'")

# ✅ GIUSTO (Django)
User.objects.filter(email=email)

# ✅ GIUSTO (raw con placeholder)
with connection.cursor() as c:
    c.execute("SELECT * FROM users WHERE email = %s", [email])
```

```go
// ❌ SBAGLIATO
db.QueryRow("SELECT id FROM users WHERE email = '" + email + "'")

// ✅ GIUSTO
db.QueryRow("SELECT id FROM users WHERE email = $1", email)
```

## Header security baseline (web)

- `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- `Content-Security-Policy: default-src 'self'; ...` (no `'unsafe-inline'` se possibile)
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: strict-origin-when-cross-origin`
- `Permissions-Policy: camera=(), microphone=(), geolocation=()` (quello che non ti serve)

## Exit criteria

- [ ] Threat model fatto (anche mentale in 5min) per ogni nuovo endpoint/modulo
- [ ] Input validati server-side con schema (Pydantic, struct tag, zod)
- [ ] Query DB parametriche, zero concat
- [ ] Authz esplicita su ogni endpoint (deny by default)
- [ ] Nessun secret/PII in log
- [ ] SCA in CI (`pip-audit`, `govulncheck`)
- [ ] Header security configurati (HSTS, CSP, X-Content-Type-Options)
- [ ] Se c'è fetch esterno: allowlist host + blocco IP privati
