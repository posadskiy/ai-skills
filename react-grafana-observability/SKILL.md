---
name: react-grafana-observability
description: >-
  Implements frontend observability (RUM, logs, traces, source maps, alerts,
  synthetic monitoring) for a React/Vite application using Grafana Faro and
  Grafana Cloud. Use when the user wants to add observability, monitoring, RUM,
  error tracking, Web Vitals, source maps, or Grafana Faro to a React app, or
  when they ask about frontend observability, Grafana Cloud setup for a React
  application, or how to track JS errors and performance in production.
compatibility: Requires React with React Router v6, Vite bundler, and a Grafana Cloud account. Nginx JSON log shipping is optional and works with any container runtime or deployment system.
metadata:
  author: posadskiy
  stack: React 19, Vite 6, React Router v6/v7, @grafana/faro-react, @grafana/faro-web-tracing, @grafana/faro-rollup-plugin, Nginx, Grafana Cloud, Grafana Alloy
---

# React + Grafana Cloud Observability

Full frontend observability: Real User Monitoring (RUM), Web Vitals, JS error tracking,
source map uploads, Nginx access log shipping, synthetic uptime checks, and alerting —
all flowing into Grafana Cloud.

**See [references/code-templates.md](references/code-templates.md) for complete copy-paste code.**
**See [references/grafana-cloud-setup.md](references/grafana-cloud-setup.md) for Grafana Cloud UI steps.**

---

## Prerequisite — ask the user for the Faro collector URL

**Before wiring code or Docker, the agent must obtain this from the user** (or guide them to copy it from Grafana):

- **Faro collector URL** — full HTTPS URL from **Grafana Cloud → Observability → Frontend → _this_ app → Instrumentation / Web SDK**, field `url` in the snippet.

  Shape:

  `https://faro-collector-prod-<region>.grafana.net/collect/<ingest-key>`

  Example shape only (do **not** reuse for another project):  
  `https://faro-collector-prod-eu-west-2.grafana.net/collect/451ee59b19fd23cfa38502c8993ecf6a`

**Why the agent must ask**

1. **Each Frontend Observability app has its own `/collect/<ingest-key>`** — that key routes browser RUM to **that** app. Copying a URL from a **different** repo or app (e.g. another product’s build script) sends data to the **wrong** Grafana app or causes **CORS** failures.
2. **`VITE_FARO_URL` is inlined at build time** — wrong URL means empty UI for the app the user cares about.
3. **CORS** is configured **per Grafana Frontend app** — production origins (`https://example.com`, `https://www.example.com`, etc.) must be allowlisted on **the same app** whose collector URL you use. See [references/grafana-cloud-setup.md](references/grafana-cloud-setup.md#cors-and-the-collector-url).

**If the user has not provided the URL yet:** ask them to create or open their Frontend app in Grafana, copy the **`url`** from the instrumentation snippet, and paste it. Do not invent or default to another project’s ingest key.

---

## What gets implemented

| Pillar | How |
|--------|-----|
| **RUM + Web Vitals** | `@grafana/faro-react` SDK in `main.tsx` |
| **JS error tracking** | Faro captures uncaught exceptions + console.error |
| **Router tracking** | `withFaroRouterInstrumentation` + `ReactIntegration` → per-route metrics |
| **Trace correlation** | `TracingInstrumentation` injects W3C `traceparent` into all fetch/XHR — links browser click to backend DB query in Tempo |
| **Source maps** | `@grafana/faro-rollup-plugin` in `vite.config.ts` — private upload at build time |
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

Add to `Dockerfile.prod` in the build stage (use `ARG` without `ENV` for the token — never stored in image):

```dockerfile
ARG VITE_FARO_URL
ARG VITE_APP_VERSION
ARG GRAFANA_OBSERVABILITY_FARO_TOKEN   # build-time only, NOT promoted to ENV

ENV VITE_FARO_URL=$VITE_FARO_URL
ENV VITE_APP_VERSION=$VITE_APP_VERSION
```

`VITE_APP_VERSION` has no hardcoded default. In `faro.ts`, fall back to the version field
from `package.json` when the env var is absent — see [references/code-templates.md](references/code-templates.md#faroTs).

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
