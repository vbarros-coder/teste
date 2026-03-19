# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Commands

```bash
npm install          # install dependencies
npm run dev          # start Vite dev server (Hono + Cloudflare adapter, hot reload)
npm run build        # production build → ./dist
npm run preview      # preview built dist via wrangler pages dev
npm run deploy       # build + deploy to Cloudflare Pages
npm run cf-typegen   # regenerate CloudflareBindings interface from wrangler.jsonc
```

There is no test framework configured. No linting or typecheck scripts are defined beyond what TypeScript provides via `tsc`.

To run the production build under PM2 (binds to `0.0.0.0:3000`):
```bash
npx pm2 start ecosystem.config.cjs
```

## Architecture

This is a **Hono** application targeting **Cloudflare Pages**, built with Vite using `@hono/vite-build/cloudflare-pages`.

### Serving model: static vs worker

Cloudflare Pages serves files from `public/` as static assets **before** the worker runs. This means:
- `public/index.html` is the file actually served to users at `GET /` — the worker's `GET /` handler is a fallback only reached when no matching static file exists.
- All `/api/*` routes are handled exclusively by the Hono worker.

There are currently **two different versions** of the SPA HTML:
- `public/index.html` (30 KB) — the file served in production via static hosting.
- `src/index.html` (35 KB) — the file the Hono worker tries to serve at `GET /`. `src/index.tsx` imports it as `./page.html?raw` (with `@ts-ignore`), but the file is named `index.html`. If the build breaks, rename `src/index.html` → `src/page.html`. These two files may have diverged; keep them in sync or consolidate.

### Server layer (`src/index.tsx`)

The Hono app is the Cloudflare Pages worker entry point:
- `GET /api/health` — returns `{ status, timestamp }` JSON. CORS is applied to all `/api/*` routes.
- `GET /` — serves the SPA HTML via `./page.html?raw` (fallback; normally superseded by static `public/index.html`).

`src/renderer.tsx` defines a Hono JSX layout renderer but is not yet wired to any route — scaffolding for future JSX-rendered routes.

### Frontend SPA (`public/index.html`)

A single self-contained HTML file with vanilla JS (no build step for the frontend). All dependencies are loaded from CDN:
- **Font Awesome** – icons
- **SheetJS (`xlsx`)** – client-side Excel parsing

On page load, `init()` fetches `/dados.json` and populates the global `allData` object:

```
allData: {
  [sectorName: string]: {
    headers: string[],
    data: Array<{ Seguradora?: string, "Pontos Focais"?: string, [field: string]: string }>
  }
}
```

The UI lets users search/filter insurers across sectors. Results are grouped by insurer name, rendered as collapsible cards, with fields organized into visual groups (`FIELD_GROUPS`: SLAs, Systems, Reports, Financial, Documentation).

#### Admin mode (upload button)

The "Atualizar Planilha" upload button is hidden by default. It is revealed by:
- Visiting `?admin=addvalora` in the URL, **or**
- Clicking the header logo 5 times within 2.5 seconds.

Admin state is stored in `sessionStorage` (key `adm=1`). In admin mode, users can upload a `.xlsx` file; SheetJS parses it client-side and replaces `allData` in memory (not persisted — resets on reload).

#### Sector configuration (`SETOR_CONFIG`)

`SETOR_CONFIG` maps sector/sheet names to display labels, Font Awesome icons, and brand colors. It is defined **inline** in both `public/index.html` and `src/index.html`, and also in the separate `public/config.js`. `public/config.js` is **not loaded** by `public/index.html` (no `<script>` tag references it) — it appears to be an orphaned duplicate. If `SETOR_CONFIG` needs updating, update the inline definition in both HTML files.

### Static assets (`public/`)

| File | Purpose |
|---|---|
| `dados.json` | Main data (~241 KB). Source of truth the app reads on load. |
| `planilha.xlsx` | Source Excel file (~514 KB) used to generate `dados.json`. |
| `config.js` | Orphaned duplicate of `SETOR_CONFIG`/`FIELD_GROUPS` — not loaded by the SPA. |
| `favicon.svg` | Site icon. |
| `static/style.css` | Minimal placeholder stylesheet. |

To permanently update the data displayed in the app, replace `public/dados.json` with an updated version derived from the source `planilha.xlsx` and redeploy.

### Cloudflare bindings

All bindings in `wrangler.jsonc` (KV, R2, D1, AI, vars) are commented out. After uncommenting and configuring bindings, run `npm run cf-typegen` to regenerate the `CloudflareBindings` interface, then type the Hono app as:

```ts
const app = new Hono<{ Bindings: CloudflareBindings }>()
```

### TypeScript config notes

- `moduleResolution: "Bundler"` — required for Vite compatibility.
- `jsxImportSource: "hono/jsx"` — JSX transforms to Hono's runtime, not React.
- `strict: true` is enabled.
