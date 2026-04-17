---
name: keycloak-oidc-patterns
description: Usa questa skill quando stai integrando Keycloak (o un altro IdP OIDC) in un'app Python, Go o TypeScript — login web, API protette da Bearer JWT, refresh token, logout, gestione ruoli. Enforce Authorization Code + PKCE, validazione RS256 con JWKS, state/nonce, e evita i pitfall classici (token in localStorage, clock skew, JWKS non cachata, audience mismatch).
---

# Keycloak / OIDC Patterns

## Principio

**OIDC non si improvvisa.** Le scelte di default sbagliate (implicit flow, JWT senza validazione audience, token in localStorage) sono vulnerabilità, non dettagli.

Regole cardine:
1. **Authorization Code + PKCE** per ogni client (anche confidential)
2. **Validazione JWT completa**: firma RS256 via JWKS + `iss` + `aud` + `exp` + `nbf`
3. **State e nonce sempre**, generati crypto-random e verificati al callback
4. **Token in cookie HttpOnly + Secure + SameSite**, mai in localStorage
5. **Logout = revoca refresh token + RP-initiated logout**, non solo clear cookie

## Processo

### 1. Setup client Keycloak
- Client type: `confidential` (backend) o `public` (SPA/mobile)
- Standard Flow (Authorization Code) **on**
- Implicit Flow **off**, Direct Grant **off** (no password flow)
- Valid Redirect URIs: lista esatta, no wildcard in produzione
- Web Origins: stesso dominio dell'app, no `*`
- PKCE: `S256` (required per public, raccomandato per confidential)

### 2. Authorization request (login)
- Genera `state` (crypto-random, min 16 byte) → salva in sessione lato server
- Genera `nonce` (crypto-random) → salva in sessione
- Genera `code_verifier` (43-128 char) + `code_challenge = BASE64URL(SHA256(verifier))`
- Redirect a `/protocol/openid-connect/auth?...&code_challenge=...&code_challenge_method=S256&state=...&nonce=...`

### 3. Callback
- **Verifica `state` == stored state** (altrimenti CSRF)
- Token exchange con `code + code_verifier` (PKCE)
- **Valida ID token**: firma (JWKS), `iss`, `aud`, `exp`, `nbf`, **`nonce`** == stored nonce
- Salva: access token (short-lived, in memory server-side), refresh token (in sessione cifrata o DB), user info in sessione

### 4. API protection (Bearer)
- Middleware che estrae `Authorization: Bearer <jwt>`
- Valida firma via JWKS **cachata** (TTL 1-24h, refresh su `kid` sconosciuto)
- Valida `iss`, `aud`, `exp`, `nbf`
- Estrai claims in `request.user` / `ctx.User`

### 5. Refresh
- Usa refresh token prima che access scada (non dopo)
- Rotation: Keycloak rilascia un nuovo refresh ad ogni uso (se configurato) → salva il nuovo
- Se refresh fallisce → forza re-login

### 6. Logout
- Revoca refresh token: POST a `/protocol/openid-connect/logout` con `refresh_token` + `client_id`
- Redirect a `end_session_endpoint` con `id_token_hint` + `post_logout_redirect_uri`
- Pulisci cookie/sessione lato app

## Anti-rationalization

| Scusa | Realtà |
|---|---|
| "Tanto il client è trusted, PKCE non serve" | PKCE costa zero e protegge anche se il client viene compromesso. Sempre. |
| "Valido solo `exp`, il resto è ridondante" | Un token di un altro `aud` o di un altro `iss` è valido sul wire, no nel tuo contesto. Mismatch = auth bypass. |
| "Cacho JWKS in memoria per sempre" | Le chiavi ruotano. Senza refresh su `kid` ignoto, rollover Keycloak ti rompe prod. |
| "Metto il token in localStorage, è comodo" | XSS = token leak. Cookie HttpOnly Secure SameSite=Lax. Sempre. |
| "Logout = cancello il cookie" | Il refresh resta valido fino a scadenza. Un attaccante che l'ha rubato continua ad averne accesso. |

## Esempio Python (FastAPI) — middleware validazione Bearer

```python
# auth.py
import httpx, jwt
from jwt.algorithms import RSAAlgorithm
from cachetools import TTLCache

JWKS_URL = f"{KC_ISSUER}/protocol/openid-connect/certs"
_jwks_cache = TTLCache(maxsize=1, ttl=3600)

def _get_key(kid: str):
    jwks = _jwks_cache.get("jwks")
    if jwks is None or kid not in {k["kid"] for k in jwks["keys"]}:
        jwks = httpx.get(JWKS_URL, timeout=3.0).json()
        _jwks_cache["jwks"] = jwks
    for k in jwks["keys"]:
        if k["kid"] == kid:
            return RSAAlgorithm.from_jwk(k)
    raise ValueError(f"unknown kid: {kid}")

def verify_access_token(token: str) -> dict:
    header = jwt.get_unverified_header(token)
    key = _get_key(header["kid"])
    return jwt.decode(
        token, key=key, algorithms=["RS256"],
        audience=KC_AUDIENCE,       # aud claim
        issuer=KC_ISSUER,           # iss claim
        options={"require": ["exp", "iat", "iss", "aud", "sub"]},
        leeway=30,                  # clock skew tolerance
    )

# dependency
async def current_user(request: Request) -> User:
    auth = request.headers.get("authorization", "")
    if not auth.startswith("Bearer "):
        raise HTTPException(401, "missing bearer")
    try:
        claims = verify_access_token(auth[7:])
    except jwt.PyJWTError as e:
        raise HTTPException(401, f"invalid token: {e}")
    return User(id=claims["sub"], roles=claims.get("realm_access", {}).get("roles", []))
```

## Esempio Go — login flow con PKCE

```go
// state/nonce/pkce generation
func NewAuthRequest() *AuthRequest {
    return &AuthRequest{
        State:        randBase64(32),
        Nonce:        randBase64(32),
        CodeVerifier: randBase64(64),
    }
}
func (r *AuthRequest) Challenge() string {
    sum := sha256.Sum256([]byte(r.CodeVerifier))
    return base64.RawURLEncoding.EncodeToString(sum[:])
}

// handler callback
func (h *AuthHandler) Callback(w http.ResponseWriter, r *http.Request) {
    stored := h.sess.Pop(r, "auth")
    if stored == nil || r.URL.Query().Get("state") != stored.State {
        http.Error(w, "state mismatch", http.StatusBadRequest)
        return
    }
    tok, err := h.oauth.Exchange(r.Context(), r.URL.Query().Get("code"),
        oauth2.SetAuthURLParam("code_verifier", stored.CodeVerifier),
    )
    if err != nil { http.Error(w, "token exchange failed", 400); return }

    idToken, err := h.verifier.Verify(r.Context(), tok.Extra("id_token").(string))
    if err != nil { http.Error(w, "id token invalid", 400); return }
    var claims struct{ Nonce string `json:"nonce"` }
    _ = idToken.Claims(&claims)
    if claims.Nonce != stored.Nonce {
        http.Error(w, "nonce mismatch", 400); return
    }
    // set HttpOnly session cookie, store refresh token server-side
    h.sess.Login(w, r, idToken.Subject, tok.RefreshToken)
    http.Redirect(w, r, "/", http.StatusFound)
}
```

## Esempio TypeScript (SvelteKit) — hook validazione Bearer server-side

```ts
// hooks.server.ts
import { createRemoteJWKSet, jwtVerify } from 'jose';

const JWKS = createRemoteJWKSet(new URL(`${env.KC_ISSUER}/protocol/openid-connect/certs`));

export const handle = async ({ event, resolve }) => {
  const auth = event.request.headers.get('authorization') ?? '';
  if (auth.startsWith('Bearer ')) {
    try {
      const { payload } = await jwtVerify(auth.slice(7), JWKS, {
        issuer: env.KC_ISSUER,
        audience: env.KC_AUDIENCE,
        clockTolerance: '30s',
      });
      event.locals.user = { id: payload.sub, roles: payload.realm_access?.roles ?? [] };
    } catch {
      // token presente ma invalido: NON settare locals.user → gli endpoint protetti risponderanno 401
    }
  }
  return resolve(event);
};
```

## Pitfall ricorrenti

- **Clock skew** tra app server e Keycloak → usare `leeway` di 30-60s
- **`aud` mancante o sbagliato** → Keycloak spesso emette `aud: "account"` di default. Configurare mapper per aggiungere il client_id.
- **JWKS non cachata** → ogni request colpisce Keycloak. Cache TTL 1h con refresh su `kid` sconosciuto.
- **Refresh token rotation** senza salvare il nuovo → prossima refresh fallisce.
- **`SameSite=None`** senza `Secure` → browser moderni rifiutano. Usare `Lax` in same-site, `None;Secure` solo se cross-site necessario.
- **Logout RP-initiated senza `id_token_hint`** → Keycloak chiede conferma all'utente. Con hint, redirect silenzioso.

## Exit criteria

- [ ] Flow: Authorization Code + PKCE S256 (no implicit, no password grant)
- [ ] `state` e `nonce` generati crypto-random, verificati al callback
- [ ] JWT validato per: firma RS256 via JWKS, `iss`, `aud`, `exp`, `nbf`, `nonce` (per ID token)
- [ ] JWKS cachata con refresh su `kid` sconosciuto
- [ ] Token salvati in cookie HttpOnly + Secure + SameSite=Lax (mai localStorage)
- [ ] Refresh token rotation gestita (salva sempre il nuovo)
- [ ] Logout revoca il refresh + RP-initiated con `id_token_hint`
- [ ] Clock leeway 30-60s configurato
