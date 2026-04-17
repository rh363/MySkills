---
name: api-contract-first
description: Usa questa skill quando stai progettando o modificando un'API HTTP/REST o gRPC con client e server in linguaggi diversi (Go/Python/TS). Enforce il pattern contract-first — OpenAPI/Protobuf spec come single source of truth, codegen ai confini, breaking change detection via diff tool in CI, versioning disciplinato.
---

# API Contract-First

## Principio

Il contratto (OpenAPI 3.1 o Protobuf) è la **sorgente di verità**. Server e client derivano da lì via codegen. Nessun DTO scritto a mano due volte. Breaking change = deliberato, versionato, annunciato con deprecation window.

## Perché

In un backend polyglot (Go API + Python worker + TS frontend), i DTO scritti a mano in 3 linguaggi producono drift silenzioso. Il drift si manifesta come `null` dove ti aspetti stringa, campi rinominati solo su un lato, 500 in prod, tipi TS che mentono al compilatore.

## Processo

### 1. Scrivi la spec prima del codice
- OpenAPI 3.1 in `api/openapi.yaml` (o `.proto` per gRPC)
- Descrivi request/response, error schema, auth, rate limits, status code non-200
- Review con il consumer **prima** di implementare — la spec è un'API review mascherata

### 2. Genera gli stub, non scriverli
- Go server: `oapi-codegen` → handler interface
- Go client: `oapi-codegen` → typed client con context
- Python server: pydantic models + FastAPI route signature dalla spec
- Python client: `openapi-python-client`
- TS client: `openapi-typescript` + `openapi-fetch`
- **Mai modificare a mano i file generati**: wrappa, non editare

### 3. Validate al boundary
- Request validation: il codegen include schema validation (es. FastAPI nativo, middleware in Go)
- Response validation: almeno in dev/staging, valida anche l'output contro la spec (contract test)
- Rifiuta payload non conformi con 400 + error schema documentato nella spec stessa

### 4. Breaking change detection in CI
- `oasdiff breaking old.yaml new.yaml --fail-on ERR` in pipeline
- Breaking consapevole → nuovo URI versionato (`/v2/...`) + deprecation header (`Sunset:`, `Deprecation:`) su v1
- Per gRPC: `buf breaking` contro main branch

### 5. Versioning
- URI versioning per breaking (`/v1/` → `/v2/`)
- Additive (nuovo campo opzionale, nuovo endpoint) = minor, nessun bump version path
- Rimuovere campi: deprecation window ≥ 90 giorni, header `Deprecation: true`, log server-side su utilizzi residui

## Anti-rationalization

| Scusa | Risposta |
|---|---|
| "Il frontend usa solo 2 campi, scrivo il tipo a mano" | Ogni diff produce drift silente. Gen dal contratto, o scrivi la spec. |
| "È un endpoint interno, non serve spec" | Interno = consumer interni multipli. Stesso drift. |
| "La spec la aggiungo dopo" | Non la aggiungi mai. Spec-first, o niente spec. |
| "Il codegen produce codice brutto" | Non editarlo, wrappalo. Il brutto è contenuto e rigenerabile. |
| "Ho fretta, modifico solo il server" | Il client TS ora mente al compilatore. Regressione garantita. |

## Esempio: Go server + TS client

**Spec** (`api/openapi.yaml`):

```yaml
paths:
  /users/{id}:
    get:
      operationId: getUser
      parameters:
        - name: id
          in: path
          required: true
          schema: { type: string, format: uuid }
      responses:
        '200':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/User' }
        '404':
          content:
            application/json:
              schema: { $ref: '#/components/schemas/ApiError' }
components:
  schemas:
    User:
      type: object
      required: [id, email]
      properties:
        id: { type: string, format: uuid }
        email: { type: string, format: email }
        deletedAt: { type: string, format: date-time, nullable: true }
    ApiError:
      type: object
      required: [code, message]
      properties:
        code: { type: string }
        message: { type: string }
```

**Go server (generato, interfaccia)**:

```go
//go:generate oapi-codegen -config cfg.yaml openapi.yaml

type ServerInterface interface {
    GetUser(w http.ResponseWriter, r *http.Request, id openapi_types.UUID)
}

// Implementi ServerInterface — tipi forti, request già parsato
```

**TS client (generato)**:

```ts
import createClient from 'openapi-fetch';
import type { paths } from './openapi.d.ts';

const client = createClient<paths>({ baseUrl: '/api' });

const { data, error } = await client.GET('/users/{id}', {
  params: { path: { id: userId } }
});
// data: User | undefined, error: ApiError | undefined
```

## Esempio: breaking change detection in CI

```yaml
# .github/workflows/api.yml
- name: Check breaking changes
  run: |
    oasdiff breaking \
      https://raw.githubusercontent.com/org/repo/main/api/openapi.yaml \
      api/openapi.yaml \
      --fail-on ERR
```

Per gRPC/Protobuf:

```yaml
- run: buf breaking --against '.git#branch=main'
```

## Esempio: contract test lato server (Python)

```python
# tests/contract/test_users.py
from openapi_core import OpenAPI
from openapi_core.contrib.requests import RequestsOpenAPIRequest, RequestsOpenAPIResponse

spec = OpenAPI.from_file_path("api/openapi.yaml")

def test_get_user_matches_spec(client, user):
    resp = client.get(f"/users/{user.id}")
    spec.validate_response(
        RequestsOpenAPIRequest(resp.request),
        RequestsOpenAPIResponse(resp),
    )  # fallisce se response diverge dalla spec
```

## Exit criteria

- [ ] Spec committata in repo, review approvata dal consumer
- [ ] Codegen in `make generate` (o script equivalente), stub NON editati a mano
- [ ] Request validation attiva su tutti gli endpoint
- [ ] `oasdiff` (o `buf breaking`) in CI blocca breaking non intenzionali
- [ ] Breaking = nuovo URI versionato + deprecation header + sunset date documentata
- [ ] Almeno un contract test verde in suite
