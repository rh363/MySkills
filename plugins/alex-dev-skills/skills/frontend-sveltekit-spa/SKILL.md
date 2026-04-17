---
name: frontend-sveltekit-spa
description: Usa questa skill quando lavori su frontend SvelteKit SPA (Svelte 5 su progetti nuovi, Svelte 4 su legacy). Copre layout $lib/svelte/page/feature, pages come entry point pulite, config runtime con Zod + $env/dynamic/public, API client fetch-based con Zod validation, stores + createPersistedStore, CSS Tailwind + @seeweb/svelte-ui, auth oidc-spa + Keycloak, i18n Paraglide, icone @lucide/svelte + SVG custom, e architettura micro-frontend iframe con postrobot. Stack-specific — frontend e backend in repo separati sempre.
---

# Frontend — SvelteKit SPA

## Stack

- **Framework**: SvelteKit con `@sveltejs/adapter-node`
- **Svelte version**: Svelte 5 su progetti nuovi (runes), Svelte 4 su legacy (stores classici) — **segui lo stile del progetto esistente**
- **Language**: TypeScript always
- **Mode**: SPA (`ssr = false`, `prerender = false`)
- **CSS**: Tailwind CSS, classi inline preferite
- **UI library**: `@seeweb/svelte-ui` (no docs online, leggi gli export dal package)
- **Icons**: `@lucide/svelte` + custom SVG wrappati come componenti Svelte
- **Auth**: `oidc-spa` con Keycloak
- **i18n**: Paraglide (progetti nuovi)
- **Validation**: Zod
- **Micro-frontend**: iframe-based con `post-robot` per comunicazione
- **Repo policy**: frontend e backend **sempre in repository separati**

## Project structure

```
src/
├── app.css                              # Tailwind + @seeweb/svelte-ui theme
├── app.html
├── lib/
│   ├── index.ts                         # Lib barrel export
│   ├── config.ts                        # Zod schema + $env/dynamic/public
│   ├── auth/
│   │   └── oidc/
│   │       ├── helpers.ts               # getOidc(), getTokens(), logout()
│   │       ├── schemas.ts               # Zod token schema
│   │       └── types.ts
│   ├── <api-client>/                    # Una cartella per backend service
│   │   ├── client.ts                    # Classe fetch-based
│   │   ├── schemas.ts                   # Zod response schemas
│   │   ├── types.ts
│   │   └── index.ts
│   ├── stores/
│   │   ├── config.ts                    # App config store
│   │   ├── user.ts                      # User claims store
│   │   ├── oidc.ts                      # OIDC instance store
│   │   ├── persisted.ts                 # createPersistedStore helper
│   │   └── <domain>.ts
│   ├── dom/
│   │   └── helpers.ts
│   ├── effects/                         # Svelte actions
│   │   └── tooltip.ts
│   ├── svelte/
│   │   ├── component/                   # Componenti globali condivisi
│   │   │   ├── Header/
│   │   │   │   ├── Header.svelte
│   │   │   │   ├── Logo.svelte
│   │   │   │   └── UserSettings.svelte
│   │   │   ├── SideBar/
│   │   │   │   ├── SideBar.svelte
│   │   │   │   └── types.d.ts
│   │   │   └── index.ts                 # Barrel export
│   │   ├── layout/
│   │   │   └── Layout.svelte
│   │   ├── iframe/                      # Micro-frontend loaders
│   │   │   ├── IframeLoader.svelte
│   │   │   └── index.ts
│   │   └── page/                        # Page-level: una cartella per feature
│   │       └── <feature>/
│   │           ├── <Feature>.svelte     # Main component
│   │           ├── components/          # Sub-componenti solo di questa feature
│   │           │   └── <Component>.svelte
│   │           ├── composables/         # Runes Svelte 5
│   │           │   └── use<Name>.svelte.ts
│   │           └── index.ts             # Barrel export
│   ├── svg/                             # SVG custom come componenti
│   │   ├── brand/
│   │   │   └── <BrandLogo>.svelte
│   │   └── ui/
│   │       └── <Icon>.svelte
│   └── paraglide/                       # Auto-generato i18n (Paraglide)
│       ├── messages/
│       ├── runtime.js
│       └── server.js
├── routes/
│   ├── +layout.ts                       # OIDC init, user store setup
│   ├── +layout.svelte                   # Layout wrapper
│   ├── +page.svelte                     # Entry point pulita → delega a $lib/svelte/page/
│   └── <route>/
│       └── +page.svelte                 # Entry point pulita
└── static/
```

## Principi chiave

### Pages sono entry point pulite

I file `+page.svelte` devono essere **minimali**: importano e renderizzano il componente page da `$lib/svelte/page/`. Tutta la logica, sub-componenti e composables stanno lì dentro.

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import { Home } from '$lib/svelte/page/home';
</script>

<Home />
```

### Page component organization

Ogni feature dentro `$lib/svelte/page/` segue lo stesso layout:

```
$lib/svelte/page/<feature>/
├── <Feature>.svelte              # Componente principale
├── components/                   # Sub-componenti (solo se servono)
│   └── <SubComponent>.svelte
├── composables/                  # Reactive logic (runes Svelte 5)
│   └── use<Name>.svelte.ts
└── index.ts                      # Barrel export
```

### Barrel exports

Ogni cartella con più file ha un `index.ts` che esporta la public API. Permette import puliti dal consumer:

```typescript
// $lib/svelte/page/home/index.ts
export { default as Home } from './Home.svelte';
```

## Config — Zod + dynamic env

Config validata a runtime con Zod, usando `$env/dynamic/public` così le env var si cambiano sul server **senza rebuild**. Questo richiede il **node adapter**, non lo static.

```typescript
// $lib/config.ts
import { env as publicEnv } from '$env/dynamic/public';
import { z } from 'zod';

export const ConfigSchema = z.object({
  oidc: z.object({
    clientId: z.string(),
    issuer: z.string(),
    baseUrl: z.string(),
    scopes: z.string().array().optional(),
  }),
  api: z.object({
    baseUrl: z.url(),
  }),
});

export type Config = z.infer<typeof ConfigSchema>;

export const getConfig = () =>
  ConfigSchema.parse({
    oidc: {
      clientId: publicEnv.PUBLIC_OIDC_CLIENT_ID,
      issuer: publicEnv.PUBLIC_OIDC_ISSUER,
      baseUrl: publicEnv.PUBLIC_OIDC_BASE_URL,
      scopes: publicEnv.PUBLIC_OIDC_SCOPES?.split(' '),
    },
    api: {
      baseUrl: publicEnv.PUBLIC_API_BASE_URL,
    },
  });
```

## SvelteKit config — SPA mode

```typescript
// svelte.config.js
import adapter from '@sveltejs/adapter-node';

export default {
  kit: { adapter: adapter() },
};
```

```typescript
// src/routes/+layout.ts
export const ssr = false;
export const prerender = false;
```

## Stores

Svelte 4 usa `writable/readable/derived`. Svelte 5 preferisce `$state` in runes, ma gli store sono ancora validi per stato globale condiviso cross-componente.

### `createPersistedStore` — store con localStorage

```typescript
// $lib/stores/persisted.ts
import { browser } from '$app/environment';
import { writable, type Writable } from 'svelte/store';

export function createPersistedStore<T>(key: string, defaultValue: T): Writable<T> {
  const initial =
    browser && localStorage.getItem(key) != null
      ? (JSON.parse(localStorage.getItem(key)!) as T)
      : defaultValue;

  const store = writable<T>(initial);

  if (browser) {
    store.subscribe((value) => {
      if (value === null) localStorage.removeItem(key);
      else localStorage.setItem(key, JSON.stringify(value));
    });
  }

  return store;
}
```

## API clients

Una classe per backend service. **Fetch-based**, nessuna libreria esterna (axios/ky). Zod **sempre** sulle response.

```typescript
// $lib/<service>/client.ts
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string(),
  email: z.string(),
  username: z.string(),
});

const UsersResponse = z.object({ users: z.array(UserSchema) });

class ApiClient {
  private token: string;
  private url: string;

  constructor(options: { token: string; url: string }) {
    this.token = options.token;
    this.url = options.url;
  }

  async fetchUsers() {
    const resp = await this.get(this.buildUrl('/v1/users/'));

    if (!resp.ok) {
      throw new Error(`Failed to fetch users: ${resp.status}: ${await resp.text()}`);
    }
    if (!resp.headers.get('content-type')?.startsWith('application/json')) {
      throw new Error(`Expected JSON, got: ${resp.headers.get('content-type')}`);
    }

    return UsersResponse.parse(await resp.json()).users;
  }

  private async get(url: URL) {
    return await fetch(url, { method: 'GET', headers: this.authHeaders() });
  }

  private async post(url: URL, body: unknown) {
    return await fetch(url, {
      method: 'POST',
      headers: { ...this.authHeaders(), 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
  }

  private authHeaders() {
    return { Authorization: `Bearer ${this.token}` };
  }

  private buildUrl(path: string, params: Record<string, string> = {}) {
    const url = new URL(`${this.url.replace(/\/+$/, '')}/${path.replace(/^\/+/, '')}`);
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, v));
    return url;
  }
}

export default ApiClient;
```

Pattern chiave:
- Costruttore prende `{ token, url }` — iniettabile, testabile
- Check `resp.ok` + content-type **prima** del parse
- `Zod.parse` sempre — niente `resp.json() as MyType`
- `buildUrl` helper per URL con params
- Una classe per backend service, **non** singoli helper function

## CSS — Tailwind

```css
/* src/app.css */
@import 'tailwindcss';
@import '@seeweb/svelte-ui/theme.css';

@source '../node_modules/@seeweb/svelte-ui/dist';
```

- **Classi Tailwind inline** nel template, preferito
- Legacy: se il progetto ha CSS in file separati, segui lo stile esistente, non rifattorizzare
- Scorciatoie: `cn()` helper per combinare classi con conditionals

## Auth — `oidc-spa` + Keycloak

`$lib/auth/oidc/helpers.ts` espone:
- `getOidc()` — istanza singleton, init via `createOidc({ ... })` con config da `$lib/config`
- `getTokens()` — ritorna access/id token + claims; fa auto-refresh
- `logout(redirectTo)` — chiama `oidc.logout()` pulendo session

`+layout.ts` chiama `getOidc()` in `load`, popola `userStore` con i claims. Protezione route = redirect a login se `!tokens`.

## i18n — Paraglide

Solo progetti nuovi. File messaggi in `messages/<locale>.json`, compilati da Paraglide in `$lib/paraglide/` (auto-generato, **gitignorato**, rigenerato in build).

```svelte
<script lang="ts">
  import * as m from '$lib/paraglide/messages';
</script>

<h1>{m.welcome_title()}</h1>
```

## Icons

- `@lucide/svelte` per icone standard
- SVG custom wrappati come componenti Svelte in `$lib/svg/` (mai `<img src="...svg">` né inline SVG ripetuti)

```svelte
<script lang="ts">
  import { User } from '@lucide/svelte';
  import { Logo } from '$lib/svg/brand';
</script>

<User size={20} />
<Logo />
```

## Micro-frontend — iframe + postrobot

Pattern per split di app grandi in sotto-app indipendenti.

**Parent app**: gestisce auth Keycloak iniziale via `oidc-spa`, ospita iframe figli. Ogni figlio gestisce il proprio client Keycloak, ma condivide lo stesso realm.

**Child app**: è un'app SvelteKit standalone, caricata via iframe. Riceve messaggi di startup dal parent tramite `post-robot`.

**Communication**: `post-robot` (fork di `post-message` di PayPal) per request/response cross-iframe tipizzate.

### `IframeLoader.svelte`

```svelte
<script lang="ts">
  import postRobot from 'post-robot';

  export let src: string;
  export let startMessage: string;
  export let startPayload: Record<string, unknown> = {};

  let iframe: HTMLIFrameElement;

  async function handleLoad() {
    await postRobot.send(iframe.contentWindow!, startMessage, startPayload);
  }
</script>

<iframe
  bind:this={iframe}
  {src}
  on:load={handleLoad}
  class="w-full h-full border-0"
  title="micro-frontend"
/>
```

### Child app — listener

```typescript
// child: src/lib/iframe/listener.ts
import postRobot from 'post-robot';

postRobot.on('app-start', (event) => {
  const { token, locale } = event.data as { token: string; locale: string };
  // Popola store, avvia routing
  return { ack: true };
});
```

### Convenzioni micro-frontend

- Ogni child ha il **suo proprio Keycloak client** (stesso realm del parent)
- Il parent **non passa il token** via postMessage se può evitarlo: il child rifà il flow auth con `oidc-spa` silent login
- Message name convention: `<child-app>:<event>` (es. `billing:start`, `billing:refresh`)
- Request/response tipizzate con schema Zod condiviso via package interno, non duplicato

## Convenzioni (riassunto)

| Area | Path | Note |
|---|---|---|
| Page feature | `$lib/svelte/page/<feature>/` | Main + `components/` + `composables/` + `index.ts` |
| Shared component | `$lib/svelte/component/<Name>/` | Header, SideBar, etc. con barrel export |
| Layout | `$lib/svelte/layout/Layout.svelte` | — |
| Iframe loader | `$lib/svelte/iframe/` | Micro-frontend |
| Store | `$lib/stores/<name>.ts` | `createPersistedStore` per localStorage |
| Auth | `$lib/auth/oidc/` | `oidc-spa` wrappers |
| API client | `$lib/<service>/client.ts` | Classe fetch-based con Zod |
| SVG | `$lib/svg/{brand,ui}/<Name>.svelte` | Componenti |
| Effects | `$lib/effects/<name>.ts` | Svelte actions |
| i18n | `$lib/paraglide/` | Auto-generato, gitignorato |
| Composable | `<feature>/composables/use<Name>.svelte.ts` | Svelte 5 runes |
| Route | `src/routes/<path>/+page.svelte` | Solo entry point: importa da `$lib/svelte/page/` |

## Anti-pattern

- `+page.svelte` con logica di business o fetch diretto → **spostare in `$lib/svelte/page/<feature>/`**
- Fetch inline nei componenti → **usare un API client class**
- `resp.json() as MyType` senza Zod → drift garantito col backend
- Libreria HTTP esterna (axios, ky) → fetch basta, meno peso
- `$env/static/public` → rebuild ad ogni cambio env. **Usa `$env/dynamic/public`** + node adapter
- Static adapter invece del node adapter → perdi il runtime config dinamico
- Token passato via postMessage al child micro-frontend → preferire silent login separato
- i18n messages inline nei componenti quando Paraglide è configurato → usare `m.xxx()`
- SSR non disabilitato in `+layout.ts` → l'app è SPA, non ibrida

## Exit criteria

- [ ] `svelte.config.js` usa `adapter-node`, `+layout.ts` ha `ssr = false` + `prerender = false`
- [ ] `$lib/config.ts` usa Zod + `$env/dynamic/public`
- [ ] Tutti i `+page.svelte` sono entry point pulite che importano da `$lib/svelte/page/`
- [ ] Ogni feature in `$lib/svelte/page/<feature>/` ha `index.ts` con barrel export
- [ ] API client: una classe per backend service, fetch-based, Zod su ogni response
- [ ] Stores persistenti usano `createPersistedStore`
- [ ] `oidc-spa` inizializzato in `+layout.ts`, token via `getTokens()`
- [ ] Tailwind + `@seeweb/svelte-ui` theme importati in `app.css`
- [ ] Icone: `@lucide/svelte` per standard, componenti in `$lib/svg/` per custom
- [ ] Paraglide configurato (progetti nuovi), `$lib/paraglide/` gitignorato
- [ ] Se micro-frontend: iframe loader in `$lib/svelte/iframe/`, postrobot per messaging, child gestisce auth propria
- [ ] Frontend e backend in repo separati
