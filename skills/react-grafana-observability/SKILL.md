---
name: react-grafana-observability
description: >-
  Implements frontend observability (RUM, logs, traces, source maps, alerts,
  synthetic monitoring) for React apps using Grafana Faro and Grafana Cloud.
  Covers Vite + React Router and Next.js (App Router / static export). Use when
  the user wants observability, RUM, Web Vitals, source maps, or Grafana Faro,
  or asks about frontend observability / Docker + Faro gotchas.
compatibility: >-
  Primary path: React + Vite + React Router v6. Alternate: Next.js (App Router,
  static export) + webpack Faro plugin. Grafana Cloud account required. Nginx JSON
  log shipping optional.
metadata:
  author: posadskiy
  stack: React 19, Vite 6 / Next.js 16, React Router v6/v7, @grafana/faro-react, @grafana/faro-web-sdk, @grafana/faro-web-tracing, @grafana/faro-rollup-plugin, @grafana/faro-webpack-plugin, Nginx, Grafana Cloud, Grafana Alloy
---

# React + Grafana Cloud Observability

Full frontend observability: Real User Monitoring (RUM), Web Vitals, JS error tracking,
source map uploads, Nginx access log shipping, synthetic uptime checks, and alerting —
all flowing into Grafana Cloud.

**See [references/code-templates.md](references/code-templates.md) for complete copy-paste code.**
**See [references/grafana-cloud-setup.md](references/grafana-cloud-setup.md) for Grafana Cloud UI steps.**

---

## Prerequisite — collect from the user **before** implementing this skill

**Hard stop — collector URL first.** The user must **paste the Faro collector `url` in this chat** (or explicitly confirm they are embedding that **exact** same string in-repo and paste it once for the agent to copy into code). **Do not continue** with this skill until that URL has been received: no `npm install`, no new files, no edits to `main.tsx` / `vite.config` / Docker / CI, and no “we will use `VITE_FARO_URL` later” scaffolding. A message like “implement observability” without the URL is **not** sufficient.

**Do not install packages, edit `main.tsx`, `vite.config`, Docker, or CI until the user has answered the checklist below.** If something is missing, ask for it explicitly and **wait**.

### 1. Source map upload token (optional but tell the user to decide up front)

- **`GRAFANA_OBSERVABILITY_FARO_TOKEN`** — Grafana Cloud access policy token with `sourcemaps:read` + `sourcemaps:write` (often `glc_…`). Used only at **build time** by the Faro Rollup/Webpack plugin to upload hidden source maps.
- **Ask the user:** whether they want readable stack traces in Grafana. If **yes**, they must **add this variable to CI / secrets / Docker build args** (wherever `vite build` / `next build` runs) if it is **not already** configured — then they provide the token value (or confirm it is already available to the build and the agent should only reference the variable name).
- If they **do not** want source map uploads, they should say so; the build still works and RUM runs — stacks stay minified.

### 1b. Source map uploader metadata (required only if uploads are enabled)

If the user wants **source map upload** (they said **yes** to the token / de-minified stacks), the Faro Rollup / Webpack plugin needs Grafana values **the agent cannot guess**. **Ask for all of the below before editing `vite.config` / `next.config` with `faroUploader`:**

| Value | Where to copy it |
|--------|------------------|
| **`appId`** | Same **Frontend** app → **Instrumentation / Web SDK** snippet or app settings (string / UUID Grafana shows for the app — **not** the same as inventing from the `/collect/` path). |
| **`stackId`** | Same place — numeric **stack** id for your Grafana Cloud stack (snippet or portal). |
| **`appName` (exact)** | The **App name** field in Grafana for **this** Frontend app — must match `initializeFaro({ app: { name } })` and the uploader’s `appName`. |
| **Faro API `endpoint`** | `https://faro-api-prod-<region>.grafana.net/faro/api/v1` where **`<region>` matches the collector host** (e.g. collector `…faro-collector-prod-eu-west-2…` → endpoint `…faro-api-prod-eu-west-2…`). Ask the user to **paste the full endpoint URL** from Grafana if the UI shows it; otherwise pasting the **collector URL** is enough for the agent to derive the region **only** when it follows the standard `faro-collector-prod-<region>` pattern. |

**If source map upload is off**, `appId` / `stackId` / endpoint are **not** required — omit the uploader plugin or leave it gated until the user provides them later.

### 2. Faro collector URL (required for env-based RUM)

- **Faro collector URL** — full HTTPS URL from **Grafana Cloud → Observability → Frontend → _this_ app → Instrumentation / Web SDK**, field `url` in the snippet.

  Shape:

  `https://faro-collector-prod-<region>.grafana.net/collect/<ingest-key>`

  Example shape only (do **not** reuse for another project):  
  `https://faro-collector-prod-eu-west-2.grafana.net/collect/451ee59b19fd23cfa38502c8993ecf6a`

**Ask the user** to paste that **`url`** after they have the Frontend app open. Do not invent or default to another project’s ingest key. **Receiving the URL is mandatory** — treat “I will add it in CI” without pasting the same string here as **not received**; reply asking for the URL (or the exact string to hardcode) before touching the repo.

### 3. Frontend app name — `app.name` / `appName` (required before RUM wiring)

**Ask the user** for the **exact App name** string shown in Grafana Cloud for **this** Frontend Observability application (the name they chose when creating the app).

- Use it identically in **`initializeFaro({ app: { name: … } })`** (Vite / Next) and in **`faroUploader({ appName: … })`** when source maps are enabled.
- **Do not** invent a pretty name from the repo folder — Grafana groups data by this string; a mismatch makes the UI look like the wrong app or an empty app.

**Order of operations:** collector `url` (§2) + exact **app name** (§3) are required before implementing RUM. If §1 is “yes”, also collect §1b **before** adding the uploader plugin.

**Why the agent must ask (collector URL + token workflow)**

1. **Each Frontend Observability app has its own `/collect/<ingest-key>`** — that key routes browser RUM to **that** app. Copying a URL from a **different** repo or app sends data to the **wrong** app or breaks ingestion.
2. **`VITE_FARO_URL` / `NEXT_PUBLIC_FARO_URL` is inlined at build time** — wrong URL means an empty or misleading Frontend Observability UI for this product.
3. **`GRAFANA_OBSERVABILITY_FARO_TOKEN` is build-time** — without it in the environment that runs the build, the Faro uploader plugin is skipped; the user should **add** it to CI/secrets **before** expecting de-minified errors (see Step 4 / Step 8).

### CORS and production origins (Grafana UI — not a “paste into chat” requirement)

Browsers send **cross-origin** `POST` requests from your site’s origin to the **Faro collector** host (`*.grafana.net`). The browser will block those responses unless Grafana’s **Frontend** app allowlists your site’s **Origin**.

- **Where it is set:** Grafana Cloud **Frontend** app settings / instrumentation UI for **that** app (see [references/grafana-cloud-setup.md](references/grafana-cloud-setup.md#cors-and-the-collector-url)) — not in your repo’s `.env`.
- **What the agent should do:** remind the user that after deploy they must allowlist **each** real production origin (`https://…`) on the **same** Frontend app whose collector `url` they pasted. **Do not** treat “list every production URL” as mandatory input in chat unless you are **debugging** failed `collect` requests (network tab shows CORS errors).

**After you have** the collector URL (§2), the exact Grafana **app name** (§3), their **decision on the token** (§1), and — if uploads are on — **appId**, **stackId**, and **endpoint** (§1b), choose one wiring style:

- **Env at build time** — `VITE_FARO_URL` / `NEXT_PUBLIC_FARO_URL` (documented below). Fine when CI always passes a non-empty value.
- **Hardcoded constant** — put the exact collector URL in source (e.g. `FARO_COLLECT_URL` in a client module). Valid when the user wants zero Faro-related env vars; still the **same** app-specific URL from Grafana (not copied from another product).

---

## Docker / Next.js pitfall — empty `ENV` disables Faro silently

If a Dockerfile does:

```dockerfile
ARG NEXT_PUBLIC_FARO_URL
ENV NEXT_PUBLIC_FARO_URL=$NEXT_PUBLIC_FARO_URL
```

(and the Vite equivalent `ARG VITE_FARO_URL` + `ENV VITE_FARO_URL=$VITE_FARO_URL`)

and the image is built **without** that `--build-arg`, the variable becomes an **empty string** at build time, **not** unset.

In JavaScript, **`"" ?? fallback` keeps `""`** (nullish coalescing only replaces `null` / `undefined`). Code that does `if (!faroUrl) return` then **never initializes Faro**, and bundlers may **tree-shake out** the default collector URL string — production images show **no `faro-collector` in `grep`** even though the app “has” observability code.

**Fix (pick one):**

1. **Do not** `ENV` optional Faro URLs — only pass `ARG` into `RUN` when non-empty, **or** omit those lines and use a **hardcoded** collector URL in code (simplest for Docker-only deploys).
2. If you keep env-based URLs, treat **empty string like unset** in code (`trim()` / length check before `??`), or always pass explicit `--build-arg` in CI.

**Verify after build:** `docker run --rm <image> grep -r faro-collector /usr/share/nginx/html/_next/static/chunks` or the equivalent static root — expect hits when RUM is wired for production.

---

## Next.js vs Vite (same Grafana Cloud; different wiring)

| Topic | Vite + React Router | Next.js (App Router, static export) |
|--------|---------------------|-------------------------------------|
| **Env** | `import.meta.env.VITE_FARO_*`, `import.meta.env.MODE` | `process.env.NODE_ENV`; avoid empty `NEXT_PUBLIC_*` `ENV` in Docker (see above) |
| **Init** | `initFaro()` top of `main.tsx` | `"use client"` component in root `layout.tsx` (e.g. `FrontendObservability`) |
| **Router / views** | `withFaroRouterInstrumentation` + `ReactIntegration` | `usePathname` + `faro.api.setView({ name })` (no React Router in App Router) |
| **Packages** | `@grafana/faro-react` + `faro-web-tracing` | Often `@grafana/faro-web-sdk` + `faro-web-tracing` only |
| **Source maps** | `@grafana/faro-rollup-plugin` in `vite.config` | `@grafana/faro-webpack-plugin` in `next.config.js` → `webpack()`; **Next 16+** default prod bundler may be Turbopack — use `next build --webpack` so the plugin runs |
| **Plugin in Docker** | Rollup plugin is devDependency | Put **`@grafana/faro-webpack-plugin` in `dependencies`** if `next.config` imports it — `next.config` loads at build time and `npm ci` with `NODE_ENV=production` can skip devDependencies |

Full Next.js-oriented snippets → [references/code-templates.md](references/code-templates.md#nextjs).

---

## What gets implemented

| Pillar | How |
|--------|-----|
| **RUM + Web Vitals** | Vite: `@grafana/faro-react` in `main.tsx`. Next: `@grafana/faro-web-sdk` in a client layout component |
| **JS error tracking** | Faro captures uncaught exceptions + console.error |
| **Router tracking** | Vite: `withFaroRouterInstrumentation` + `ReactIntegration`. Next: `faro.api.setView` + `usePathname` |
| **Trace correlation** | `TracingInstrumentation` injects W3C `traceparent` into all fetch/XHR — links browser click to backend DB query in Tempo |
| **Source maps** | Vite: `@grafana/faro-rollup-plugin`. Next: `@grafana/faro-webpack-plugin` + `next build --webpack` |
| **Nginx access logs** | JSON log format → Alloy → Loki; filter by `{status=~"5.."}` |
| **Alerts** | 7 predefined Grafana Cloud alerts: Errors Count, New Errors, LCP, CLS, TTFB, FCP, INP |
| **Synthetic monitoring** | HTTP uptime checks from multiple regions |

---

## Step 1 — Install packages

```bash
npm install @grafana/faro-react @grafana/faro-web-tracing
npm install --save-dev @grafana/faro-rollup-plugin
```

Peer dependency: `@grafana/faro-react` may list an optional `react-router@7` peer while your app uses v6 — use `npm install --legacy-peer-deps` if npm errors, or align to React Router v7. See project `package.json` / lockfile.

---

## Step 2 — Create Faro initialisation module

Create `src/lib/observability/faro.ts`.
Full code → [references/code-templates.md](references/code-templates.md#faroTs).

Key points:
- Import everything from `@grafana/faro-react` (it re-exports `@grafana/faro-web-sdk`)
- Guard with `if (!faroUrl) return` — no-op in local dev when `VITE_FARO_URL` is unset
- Include `ReactIntegration` with `createReactRouterV6DataOptions({ matchRoutes })` for per-route metrics
- `sessionTracking.samplingRate: 1.0` (100%) is right for most apps — only reduce if you have very high traffic (tens of thousands of daily users) and Grafana Cloud free-tier limits become a concern

---

## Step 3 — Wire into main.tsx

Two changes to `src/main.tsx`:

```typescript
// 1. Top of file — before any React import
import { initFaro } from "@/lib/observability/faro";
initFaro();

// 2. Import and use withFaroRouterInstrumentation
import { withFaroRouterInstrumentation } from "@grafana/faro-react";

const router = withFaroRouterInstrumentation(createBrowserRouter([
  // ... your routes unchanged
]));
```

`initFaro()` must run before React renders to capture page-load performance entries.
`withFaroRouterInstrumentation` patches the router so Faro intercepts React Router v6 navigations.

---

## Step 4 — Configure Vite for source maps

Update `vite.config.ts` to:
1. Use function form `defineConfig(({ mode }) => ...)`
2. Add `build: { sourcemap: "hidden" }` — generates `.map` files but omits `sourceMappingURL` from JS bundles (source stays private)
3. Add `faroUploader` plugin guarded by `mode === "production" && !!process.env.GRAFANA_OBSERVABILITY_FARO_TOKEN`

Full code → [references/code-templates.md](references/code-templates.md#viteConfig).

---

## Step 5 — Nginx JSON access logs (containerised frontends)

Add JSON `log_format` at the top of `nginx.prod.conf` (outside the `server {}` block).
Full config → [references/code-templates.md](references/code-templates.md#nginx).

The Alloy Loki pipeline uses `stage.match { selector = "{service=\"<your-app>\"}" }` to parse
Nginx JSON separately from Java/Logstash JSON.

---

## Step 6 — Dockerfile build args

Use `ARG` for **`GRAFANA_OBSERVABILITY_FARO_TOKEN`** only on the `RUN npm run build` line (or equivalent) — **do not** `ENV` the token into the final image.

For **Vite**: if you pass `VITE_FARO_URL` / `VITE_APP_VERSION` via Docker, ensure every production build supplies **non-empty** values **or** use a **hardcoded** collector URL in `faro.ts` and drop those `ENV` lines. See **Docker / Next.js pitfall** above.

```dockerfile
ARG VITE_FARO_URL
ARG VITE_APP_VERSION
ARG GRAFANA_OBSERVABILITY_FARO_TOKEN   # build-time only, NOT promoted to ENV

ENV VITE_FARO_URL=$VITE_FARO_URL
ENV VITE_APP_VERSION=$VITE_APP_VERSION
# GRAFANA_OBSERVABILITY_FARO_TOKEN intentionally has no ENV line
```

`VITE_APP_VERSION` has no hardcoded default. In `faro.ts`, fall back to `package.json` when unset — see [references/code-templates.md](references/code-templates.md#faroTs).

**Repeatable Docker builds:** stale `out/` / `.next` layers can ship old bundles (e.g. missing Faro). Use `docker buildx build --no-cache` in your build script when debugging “deployed tag is new but JS is old”.

---

## Step 7 — Grafana Cloud setup

Full UI walkthrough → [references/grafana-cloud-setup.md](references/grafana-cloud-setup.md).

Summary:
1. Create Frontend Observability app → copy Faro collector URL → set as `VITE_FARO_URL`
2. Create Access Policy with `sourcemaps:read` + `sourcemaps:write` → generate `glc_` token → set as `GRAFANA_OBSERVABILITY_FARO_TOKEN`
3. Enable all 7 predefined alerts (Errors Count, New Errors, LCP, CLS, TTFB, FCP, INP)
4. Add Synthetic Monitoring HTTP check on `/` and `/health`

---

## Step 8 — Provide build variables

Use whatever build system you already have. The only requirement is that these variables
are available when `vite build` runs:

| Variable | Required | Notes |
|----------|----------|-------|
| `VITE_FARO_URL` | Yes | Faro collector URL from Grafana Cloud — not a secret, safe to hardcode in build config |
| `VITE_APP_VERSION` | No | Release tag or git SHA; falls back to `package.json` version if unset |
| `GRAFANA_OBSERVABILITY_FARO_TOKEN` | No | Source map upload token; omitting skips upload (stack traces stay minified) |

**If using Docker build args**, pass them via `--build-arg`. **If using CI/CD** (GitHub Actions,
GitLab CI, etc.), set them as environment variables or secrets before the build step.
**If building locally**, export them in your shell before running the build command.

The `GRAFANA_OBSERVABILITY_FARO_TOKEN` is optional — the build always succeeds without it.

**Forwarding the token from the host into Docker:** the daemon does not see shell exports automatically — pass `--build-arg GRAFANA_OBSERVABILITY_FARO_TOKEN="$GRAFANA_OBSERVABILITY_FARO_TOKEN"` when the variable is set (wrap in your `build-and-push.sh`).

---

## Where to see results

| What | Where in Grafana Cloud |
|------|----------------------|
| Web Vitals, errors, sessions | Frontend → Frontend Observability → your app |
| Nginx access logs | Explore → Loki → `{service="<app-name>"}` |
| Error rate filter | Loki → `{service="<app-name>"} \| status =~ "5.."` |
| Full browser→DB traces | Explore → Tempo → search by service name |
| Uptime / response time | Synthetic Monitoring → your check |
| Alerts | Alerting → Alert rules |
