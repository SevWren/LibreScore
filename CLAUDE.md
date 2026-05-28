# CLAUDE.md — LibreScore

## Project Overview

LibreScore is a **decentralized, open-source sheet music platform** built as a Progressive Web App (PWA). Its core philosophy is that all score data lives on IPFS — not a central server — and users are identified by cryptographic public keys, not usernames and passwords.

Think of it like a Git repository where files are content-addressed by hash (CIDs), but for sheet music: the "commit" is a ScorePack, the "content" is a MuseScore `.mscz` file, and the "remote" is IPFS.

**Stack:** Vue 3 · TypeScript · Ionic Vue (UI framework) · Webpack (via Vue CLI) · Jest

**License:** GPL-3.0

---

## Repository Layout

```
LibreScore/
├── SPEC/scorepack.md          # Data format specification (read this first)
├── src/
│   ├── main.ts                # Entry point — loads polyfills, mounts UI
│   ├── registerServiceWorker.ts
│   ├── loader.js              # IPFS HTTP Gateway routing workaround (ES5)
│   ├── core/
│   │   ├── indexing/          # Score catalogue & local cache
│   │   │   ├── index.ts       # IndexingInfo interface
│   │   │   ├── local-index.ts # Dexie/IndexedDB local cache
│   │   │   └── repo.ts        # Abstract Repo + RepoDagKV / RepoDagKVSharded
│   │   └── scorepack/
│   │       ├── index.ts       # ScorePack class (serialize / sign / verify)
│   │       └── load.ts        # fromCid() helper
│   ├── identity/
│   │   ├── index.ts           # Identity interface
│   │   ├── pubkey.ts          # libp2p-crypto key helpers
│   │   ├── user-profile.ts    # IPNS-resolved user profiles
│   │   └── provider/
│   │       ├── index.ts       # IdentityProvider registry
│   │       ├── metamask.ts    # MetaMask (entropy-derived ed25519)
│   │       ├── metamask-direct.ts  # MetaMask (direct secp256k1)
│   │       ├── webauthn.ts    # WebAuthn (hardware key)
│   │       └── key.ts         # Raw libp2p private key (hex / multibase)
│   ├── ipfs/
│   │   ├── init.ts            # ipfsInstance singleton (Infura + ipfs.io)
│   │   ├── fetch.ts           # ipfsFetch() for DAG-PB/UnixFS objects
│   │   └── index.ts           # Re-exports
│   ├── mscore/
│   │   ├── init.ts            # WebMscoreLoad() — loads WASM + SoundFont
│   │   ├── measures.ts        # Measures class — time↔element red-black tree
│   │   └── synthesizer.ts     # Synthesizer class — Web Audio API playback
│   ├── ui/
│   │   ├── App.vue            # Root component — injects ipfsInstance
│   │   ├── Home.vue           # Landing page with score list
│   │   ├── 404.vue
│   │   ├── index.ts           # createApp() + IonicVue + router + i18n
│   │   ├── router/index.ts    # Routes: / | /score/:cid | /*
│   │   ├── i18n/              # vue-i18n locale files (en.json is the source)
│   │   ├── mixins/            # Reusable component logic
│   │   │   ├── modal.ts
│   │   │   ├── popover.ts
│   │   │   ├── str-fmt.ts     # printTime / fmtDate / fmtUrl
│   │   │   └── user-profile.ts
│   │   ├── components/        # Shared UI atoms
│   │   │   ├── ActionItem.vue
│   │   │   ├── ActionList.vue
│   │   │   ├── AuthMethodItem.vue
│   │   │   ├── AuthModal.vue
│   │   │   ├── IpfsImg.vue
│   │   │   ├── UserChip.vue
│   │   │   └── UserProfileBox.vue
│   │   ├── scorelist/         # Home page score grid
│   │   │   ├── ScoreList.vue
│   │   │   ├── ScoreItem.vue
│   │   │   └── ScoreThumbnail.vue
│   │   ├── scoreview/         # Score detail / playback page
│   │   │   ├── ScoreView.vue
│   │   │   ├── ScoreViewMain.vue
│   │   │   ├── ScoreHeaderBar.vue
│   │   │   ├── ScoreInfo.vue
│   │   │   ├── ScorePlayback.vue
│   │   │   ├── ScoreComments.vue
│   │   │   ├── SheetView.vue
│   │   │   └── SheetHeightAdapter.vue
│   │   └── seo/               # Dynamic meta tag management
│   │       ├── index.ts
│   │       └── meta-tag.ts
│   └── utils/
│       ├── index.ts           # fetchData / sleep / isDev / getBaseUrl
│       ├── audio-ctx.ts       # Safari AudioContext polyfill
│       ├── body-polyfill.ts   # Response.prototype.body polyfill
│       ├── css.ts             # toPercentage()
│       ├── fmt.ts             # printTime / fmtDate / fmtUrl
│       └── idb-polyfill.ts    # fake-indexeddb for Firefox Private / Tor
├── public/
│   ├── index.html             # EJS template; injects loaderScript at <head>
│   ├── _redirects             # Netlify SPA fallback rule
│   └── robots.txt
├── SPEC/scorepack.md          # ScorePack format specification
├── vue.config.js              # Webpack / PWA / transpileDependencies config
├── tsconfig.json
├── babel.config.js
├── jest.config.js
└── .eslintrc
```

---

## Critical Domain Concepts

### ScorePack

The **central data structure** of the project. Think of it like a signed, content-addressed "envelope" around a MuseScore file.

- Encoded in **DAG-CBOR** (a binary, IPLD-compatible serialization — like CBOR but with native CID support).
- Stored on IPFS; identified by its CID.
- **Immutable** — revisions are linked via the `_prev` field (forming an append-only chain, like a Git commit graph).
- Contains metadata (`title`, `description`, `tags`, `source`, timestamps) plus a CID pointing to the actual `.mscz` file.
- Has a `_sig` field: a cryptographic signature over the whole pack with `_sig = null` (sign-then-embed pattern).
- Implemented in `src/core/scorepack/index.ts` as the `ScorePack` class.

Key error sentinels (defined at module top-level, not thrown inline):
```ts
ERR_SCOREPACK_INVALID, ERR_MISSING_TITLE, ERR_MISSING_SCORE_CID,
ERR_NO_SIGNATURE, ERR_PREV_SCORE_INVALID
```

### IndexingInfo

A flattened, denormalized snapshot of a score's metadata stored locally in IndexedDB (via Dexie). Think of it as the "search index entry" — it avoids re-fetching the full ScorePack from IPFS just to render a thumbnail and title in the score list.

Fields: `_repo`, `_id`, `scorepack` (CID string), `thumbnail` (CID string), `title`, `duration`, `npages`, `nparts`, `instruments[]`, `updated`, `created`.

### Repo Types

Repositories are remote IPFS DAG structures the app pulls `IndexingInfo` entries from. Two implementations exist:

- **`RepoDagKV`** — a single DAG-CBOR node containing `{ [_id]: IndexingInfoPartial }` (a flat key-value map).
- **`RepoDagKVSharded`** — a DAG-CBOR node pointing to multiple `RepoDagKV` nodes (sharded for scale).

Both extend the abstract `Repo` base class and yield batches of `IndexingInfo` through `iterate()`. Progress is checkpointed in IndexedDB via `lastKey` so re-indexing is incremental.

### Identity System

Users are identified by **cryptographic public keys**, not accounts. The `Identity` interface has exactly two methods: `publicKey()` and `sign(data)`.

Three built-in identity providers (in `src/identity/provider/`):

| Provider | Type string | How it works |
|---|---|---|
| MetaMask | `metamask` | Signs an entropy message; derives an **ed25519** key from the SHA-256 hash of the signature. Passwordless. |
| MetaMask Direct | `metamask-direct` | Uses the MetaMask account's **secp256k1** key pair directly. |
| WebAuthn | `webauthn` | Uses a hardware authenticator; derives ed25519 from `userHandle`. Requires ResidentKey (Chrome ≥ 76). |
| libp2p Key | `libp2p-key` | Import a raw private key as hex or multibase string. |

Providers expose a declarative `inputs` config (select or text fields) that drives the `AuthMethodItem.vue` UI — no hardcoded forms.

### IPFS Access

The app is a **read-only IPFS client** (no write in the current codebase). It uses a lightweight "lite" client assembled from individual API modules of `ipfs-http-client` to minimize bundle size:

```
ipfsInstance = Infura (cat, block.put) + ipfs.io (cat, block.get, dag.get, dag.resolve)
```

The routing rationale: Infura does not support IPNS resolution, so `dag.get` (which uses `dag.resolve` internally) is patched to use ipfs.io instead.

### MuseScore / WebMscore

`webmscore` is a **WebAssembly build of MuseScore** that runs in-browser. It renders score pages to images and synthesizes audio. Key points:

- Loading is expensive — `WebMscoreLoad()` kicks off SoundFont and CJK font fetches concurrently with the WASM init.
- SoundFont (`.sf3`) and fonts are fetched from jsDelivr CDN with `cache: 'force-cache'` and are also pre-cached by the service worker.
- The `Synthesizer` class wraps the Web Audio API. It uses a red-black tree to cache `AudioFragment`s by start time, enabling seek-without-re-synth. Audio is synthesized in configurable batch sizes (default 8 frames × 512 = ~94ms per fragment).
- `Measures` uses two red-black trees to translate between playback time (ms) and score element IDs for the synchronized cursor.

---

## Development Commands

```bash
# Install dependencies
npm install

# Start dev server (hot reload, development mode)
npm run serve

# Start dev server in production mode (service worker active)
npm run serve:prod

# Build for production
npm run build

# Build in development mode (sourcemaps, no minification)
npm run build:dev

# Run unit tests
npm test   # (jest via @vue/cli-plugin-unit-jest)

# Check i18n coverage (untranslated keys)
npm run i18n:report

# Create ipfs-404.html alias in dist/ (required for IPFS SPA hosting)
npm run link:ipfs-spa

# Clean build output
npm run clean
```

---

## Build & Configuration Notes

### vue.config.js — Key Behaviors

- **`publicPath: './'`** — relative paths, required for IPFS gateway hosting.
- **`transpileDependencies`** — a regex and list of package names that must be transpiled from ES modules. Adding a new IPFS/libp2p dependency usually means adding it here if you see syntax errors in the bundle.
- **PWA** — `@vue/cli-plugin-pwa` (Workbox). Max cache size is 10 MiB per file. IPFS CDN URLs (versioned jsdelivr) and IPFS API read endpoints are runtime-cached with `CacheFirst`.
- **`loaderScript`** — the content of `src/loader.js` is minified at build time and injected directly into `<head>` via the HTML webpack plugin. It fixes the `<base href>` for IPFS gateway paths before any SPA routing occurs.
- **`landingScript`** — a small inline script that checks for `WebAssembly` support and shows a loading message or an "update your browser" notice.

### TypeScript

- Target: `es2019`, module: `esnext`, `strictNullChecks: true`, `noImplicitThis: true`.
- Path alias: `@/*` → `src/*`.
- `resolveJsonModule: true` (used by `vue.config.js` to read `package.json` for `VUE_APP_VERSION`).
- `.vue` files are typed via the ambient declaration in `src/global.d.ts`.

### ESLint

Extends `@vue/standard`, `@vue/typescript/recommended`, `plugin:@typescript-eslint/recommended-requiring-type-checking`, and `plugin:vue/vue3-essential`.

Notable disabled rules:
- `@typescript-eslint/no-explicit-any` — off (used extensively in Vue component props for CID types)
- `@typescript-eslint/no-unsafe-*` — mostly off
- `@typescript-eslint/no-floating-promises` — **warn** (not error; be aware that async calls dropped without `void` or `await` will produce warnings)

Run linting: `npx vue-cli-service lint` (not in package.json scripts; add if needed).

---

## Architecture Patterns

### IPFS Injection via `provide/inject`

`App.vue` provides `ipfsInstance` under the key `'ipfs'`. Components that need IPFS (e.g. `ScoreThumbnail.vue`, `UserProfileMixin`) inject it via `inject: ['ipfs']`. Do not import `ipfsInstance` directly in components — use injection so the instance can be swapped in tests.

### Async Component Loading

`ScoreList` is loaded asynchronously in `Home.vue` via `defineAsyncComponent`. Heavy modules (`webmscore`, `ScorePack`, `pubkey`) use dynamic `import()` at the call site for code splitting. Follow this pattern for any new heavy dependency.

### Vue Mixins

The project uses the Options API mixin pattern (not Composition API composables). New reusable logic should follow the existing mixin style in `src/ui/mixins/` unless you are intentionally modernising to `<script setup>`.

### ActionList / ActionItem Pattern

The score detail view exposes file download and other side effects through a declarative `{ [groupName]: Action[] }` structure passed to `ActionList.vue`. Add new score actions there rather than embedding buttons directly in view components.

### Repo Iterator Pattern

`Repo.iterate()` is an `AsyncGenerator` that yields `string[]` (IDs of entries added to the local index). Callers must `for await` over it; simply calling `.collect()` will consume the entire remote repo into the local index synchronously. Be careful with large repos.

---

## Environment Variables

Set at build time in `vue.config.js` (read from there, not `.env` files by default):

| Variable | Value | Where used |
|---|---|---|
| `VUE_APP_NAME` | `'LibreScore'` | Page title, PWA name, auth message |
| `VUE_APP_DESC` | `'Sheet music. Free. Forever.'` | Meta description |
| `VUE_APP_ID` | `'librescore.org'` | WebAuthn `rpId`, MetaMask auth message |
| `VUE_APP_VERSION` | from `package.json` | Available for display |
| `BASE_URL` | Set by Vue CLI | `getBaseUrl()` in `src/utils/index.ts` |

---

## Testing

- **Framework:** Jest via `@vue/cli-plugin-unit-jest` with the TypeScript + Babel preset.
- **Vue Test Utils:** `@vue/test-utils` v2.
- **IndexedDB:** `fake-indexeddb` shim is available (already listed in devDependencies).
- Test files live in `tests/` (included in `tsconfig.json`).
- No existing test files are visible in the digest — the testing infrastructure is wired but tests have not been written yet.

---

## Known Limitations & WIP Areas

- **Comments** (`ScoreComments.vue`) — the submit button is hardcoded `disabled`. Comment submission is not implemented.
- **`copyright` and `tags`** fields in the ScorePack spec are marked `*WIP*`.
- **`RepoDagKVSharded`** overrides `type` with a `@ts-expect-error` comment — the type override is intentional but fragile.
- **`SORTING`** in `local-index.ts` only has `'latest'`; the `default` branch in `_getQuery` throws. Any new sort order must be added to the union type and the switch statement together.
- **Identity provider list is mutable** — `registerProviders()` pushes to the module-level `PROVIDERS` array. Be aware this is not thread-safe across concurrent module loads, though in practice browsers are single-threaded.
- **User profile custom data** is fetched from IPNS (slow, ~30s timeout on bad networks). The `resolveUserProfile` generator yields the generic profile first for immediate display, then the enriched profile when IPNS resolves.

---

## Adding a New Identity Provider

1. Create `src/identity/provider/my-provider.ts` implementing the `IdentityProvider` interface.
2. Export a named constant (not a class).
3. Import and add it to the `PROVIDERS` array in `src/identity/provider/index.ts`.
4. Add a logo image under `src/identity/provider/img/` and import it in your provider file.
5. The `AuthMethodItem.vue` and `AuthModal.vue` components will pick it up automatically from the registry.

## Adding a New Route

Routes are in `src/ui/router/index.ts`. The catch-all `/:catchAll(.*)` must remain last. After adding a route, update `updatePageMetadata()` calls if the new page needs custom SEO metadata.

## Adding Translations

1. Add keys to `src/ui/i18n/en.json` (the source of truth).
2. Duplicate keys into any other locale JSON files under `src/ui/i18n/`.
3. Run `npm run i18n:report` to find missing keys across locales.
