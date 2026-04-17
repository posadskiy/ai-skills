# Code Templates

Complete, copy-paste ready code for every file that needs to change.

---

## Faro collector URL (read first)

**Do not copy a collector URL from another repository or product.** Each Grafana Frontend Observability app has a unique path:

`https://faro-collector-prod-<region>.grafana.net/collect/<ingest-key>`

Use the **`url`** from **this** app’s Instrumentation snippet. The agent implementing the skill should have asked the user for that URL (or confirmed they created the app and pasted it).

After you have it, either:

- set **`VITE_FARO_URL`** at build time / Docker `--build-arg`, and use **Pattern A** below, or
- set a **`DEFAULT_FARO_COLLECT_URL`** constant in `faro.ts` to that exact string (production default) and optionally still allow `VITE_FARO_URL` to override — **Pattern B**, or
- **Pattern C (Docker-friendly)** — use **only** a hardcoded `FARO_COLLECT_URL` (and fixed `app.name`) in a client module; **no** `VITE_*` / `NEXT_PUBLIC_*` for Faro. Optional: `GRAFANA_OBSERVABILITY_FARO_TOKEN` **only** for source map upload at build time.

If the ingest key does not match the Grafana app you open in the UI, or if **CORS allowed origins** on that app do not include your site’s `https://` origin, the browser will show CORS errors and no RUM in that app.

### Anti-pattern: `ENV` optional URL without a build-arg

```dockerfile
ARG NEXT_PUBLIC_FARO_URL
ENV NEXT_PUBLIC_FARO_URL=$NEXT_PUBLIC_FARO_URL
```

If the arg is omitted, Next inlines **`""`**. Then `process.env.NEXT_PUBLIC_FARO_URL ?? default` still yields **`""`** (empty string is not `null`/`undefined`, so `??` does not use the default). Faro never starts; **`grep faro-collector` inside the image** can show nothing. Same idea for `VITE_FARO_URL` + Vite.

**Prefer:** hardcoded URL (Pattern C), or no `ENV` line for that variable, or code that treats `trim() === ""` as unset.

---

## faro.ts {#faroTs}

`src/lib/observability/faro.ts` — create this file.

```typescript
// Requires "resolveJsonModule": true in tsconfig.json (already default in Vite projects)
import { matchRoutes } from "react-router-dom";
import {
  ReactIntegration,
  createReactRouterV6DataOptions,
  faro,
  getWebInstrumentations,
  initializeFaro,
} from "@grafana/faro-react";
import { TracingInstrumentation } from "@grafana/faro-web-tracing";
import packageJson from "../../package.json";   // adjust relative path depth to match your file location

const faroUrl = import.meta.env.VITE_FARO_URL as string | undefined;

/**
 * Initialise Grafana Faro RUM.
 * Call at the very top of main.tsx, before React renders.
 * No-op when VITE_FARO_URL is unset (local dev).
 */
export function initFaro(): void {
  if (!faroUrl) {
    return;
  }

  initializeFaro({
    url: faroUrl,
    app: {
      name: "<your-app-name>",   // match the name used in Grafana Cloud
      // VITE_APP_VERSION is set at build time (git SHA, release tag, etc.).
      // Falls back to package.json version so it's never meaningless.
      version: (import.meta.env.VITE_APP_VERSION as string | undefined) ?? packageJson.version,
      environment: import.meta.env.MODE,
    },
    sessionTracking: {
      // 100% is correct for most apps. Only reduce if you have very high traffic
      // AND are hitting Grafana Cloud free-tier session limits.
      samplingRate: 1.0,
    },
    instrumentations: [
      // Errors, console.error, Web Vitals (LCP, CLS, INP), resource timings
      ...getWebInstrumentations({
        captureConsole: true,
        enablePerformanceInstrumentation: true,
      }),
      // Injects W3C traceparent into fetch/XHR → links browser trace to backend OTel spans
      new TracingInstrumentation(),
      // Tracks React Router navigations as Faro view-change events (per-route metrics)
      new ReactIntegration({
        router: createReactRouterV6DataOptions({
          matchRoutes,
        }),
      }),
    ],
  });
}

export { faro };
```

---

## Next.js — `FrontendObservability` client component {#nextjs}

For **App Router** + **`output: "export"`** (static HTML in nginx): no `main.tsx`. Use **`@grafana/faro-web-sdk`** (+ `faro-web-tracing`) in a **`"use client"`** module mounted once from `app/layout.tsx`.

```tsx
"use client";

import {
  faro,
  getWebInstrumentations,
  initializeFaro,
} from "@grafana/faro-web-sdk";
import { TracingInstrumentation } from "@grafana/faro-web-tracing";
import packageJson from "../package.json";

const FARO_COLLECT_URL = "https://faro-collector-prod-<region>.grafana.net/collect/<ingest-key>";
const FARO_APP_NAME = "<your-app-name>"; // must match Grafana + Faro webpack plugin appName

const faroUrl =
  process.env.NODE_ENV === "production" ? FARO_COLLECT_URL : undefined;

export function FrontendObservability(): null {
  if (!faroUrl || faro.api) return null;
  try {
    initializeFaro({
      url: faroUrl,
      app: {
        name: FARO_APP_NAME,
        version: packageJson.version,
        environment: process.env.NODE_ENV,
      },
      sessionTracking: { samplingRate: 1.0 },
      instrumentations: [
        ...getWebInstrumentations({
          captureConsole: true,
          enablePerformanceInstrumentation: true,
        }),
        new TracingInstrumentation(),
      ],
    });
  } catch {
    /* optional: log */
  }
  return null;
}
```

**Per-route views (optional):** small child component using `usePathname` + `faro.api.setView({ name: pathname })` in `useEffect`.

**`next.config`:** import `@grafana/faro-webpack-plugin`; gate on `process.env.GRAFANA_OBSERVABILITY_FARO_TOKEN`; set `productionBrowserSourceMaps` when uploading. **Next 16+:** run **`next build --webpack`** if the default bundler is Turbopack (otherwise custom `webpack()` is ignored).

**Dependencies:** keep **`@grafana/faro-webpack-plugin` in `dependencies`** (not only devDependencies) if `next.config` imports it — Docker / `npm ci` under production may omit devDependencies.

---

## main.tsx changes {#mainTsx}

Add two things to your existing `src/main.tsx`:

```typescript
// ── ADD AT THE VERY TOP (before all other imports) ──────────────────────────
import { initFaro } from "@/lib/observability/faro";
initFaro();

// ── ADD with your other imports ──────────────────────────────────────────────
import { withFaroRouterInstrumentation } from "@grafana/faro-react";

// ── WRAP your existing createBrowserRouter call ──────────────────────────────
const router = withFaroRouterInstrumentation(createBrowserRouter([
  // ... your existing routes unchanged ...
]));

// RouterProvider usage stays the same:
// <RouterProvider router={router} />
```

---

## vite.config.ts {#viteConfig}

Replace your existing `vite.config.ts`:

```typescript
import path from "path";
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";
import faroUploader from "@grafana/faro-rollup-plugin";

export default defineConfig(({ mode }) => ({
  plugins: [
    react(),
    // Uploads source maps only in production builds when token is present.
    // Without the token the build still succeeds — source maps just aren't uploaded.
    mode === "production" && !!process.env.GRAFANA_OBSERVABILITY_FARO_TOKEN
      ? faroUploader({
          appName: "<your-app-name>",   // match Grafana Cloud app name
          endpoint: "https://faro-api-prod-eu-west-2.grafana.net/faro/api/v1",
          appId: "<your-app-id>",       // from Grafana Cloud → Frontend Observability → app settings
          stackId: "<your-stack-id>",   // numeric ID shown in the Grafana Cloud portal
          verbose: true,
          apiKey: process.env.GRAFANA_OBSERVABILITY_FARO_TOKEN,
          gzipContents: true,
        })
      : false,
  ],
  build: {
    // 'hidden' generates .map files (faroUploader reads them) but omits
    // sourceMappingURL comments → browsers never download your source.
    sourcemap: "hidden",
  },
  resolve: {
    alias: {
      "@": path.resolve(__dirname, "./src"),
    },
  },
  server: {
    port: 3000,
  },
}));
```

> **Where to find `appId` and `stackId`:** Grafana Cloud → Frontend Observability → your app →
> the instrumentation snippet Grafana Cloud generates shows both values.

---

## nginx.prod.conf {#nginx}

Add the `log_format` block **outside and above** the `server {}` block:

```nginx
log_format json_combined escape=json
    '{'
        '"time":"$time_iso8601",'
        '"method":"$request_method",'
        '"uri":"$request_uri",'
        '"status":$status,'
        '"bytes":$body_bytes_sent,'
        '"duration":$request_time,'
        '"referrer":"$http_referer",'
        '"user_agent":"$http_user_agent",'
        '"x_forwarded_for":"$http_x_forwarded_for"'
    '}';

server {
    listen 3000;
    server_name localhost;
    root /usr/share/nginx/html;
    index index.html;

    access_log /dev/stdout json_combined;
    error_log  /dev/stderr warn;

    # ... your existing gzip, location blocks, etc. ...

    location /health {
        access_log off;   # suppress health probe noise at the source
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

---

## Dockerfile.prod changes {#dockerfile}

Add to the build stage (after existing `ARG`/`ENV` lines):

```dockerfile
ARG VITE_FARO_URL
ARG VITE_APP_VERSION=dev
# Build-time only — NOT promoted to ENV so it is never stored in the image.
ARG GRAFANA_OBSERVABILITY_FARO_TOKEN

ENV VITE_FARO_URL=$VITE_FARO_URL
ENV VITE_APP_VERSION=$VITE_APP_VERSION
# GRAFANA_OBSERVABILITY_FARO_TOKEN intentionally has no ENV line
```

**Minimal Next.js / hardcoded-URL Dockerfile** (only source-map token optional):

```dockerfile
ARG GRAFANA_OBSERVABILITY_FARO_TOKEN
RUN GRAFANA_OBSERVABILITY_FARO_TOKEN="${GRAFANA_OBSERVABILITY_FARO_TOKEN}" npm run build
```

Do **not** add `ENV NEXT_PUBLIC_FARO_URL=` (or similar) unless CI always passes a non-empty `--build-arg` — see anti-pattern above.

---

## build-and-push.sh changes {#buildScript}

Add to your Docker build command:

```bash
# Optional warning if token is missing
if [ -z "$GRAFANA_OBSERVABILITY_FARO_TOKEN" ]; then
  echo "⚠️  GRAFANA_OBSERVABILITY_FARO_TOKEN not set — source maps will NOT be uploaded."
  echo "   Stack traces will show minified code in Grafana Cloud."
fi

docker buildx build --platform linux/arm64 \
  # ... your existing --build-arg lines ... \
  --build-arg VITE_FARO_URL=https://faro-collector-prod-eu-west-2.grafana.net/collect/<app-id> \
  --build-arg VITE_APP_VERSION="$VERSION" \
  --build-arg GRAFANA_OBSERVABILITY_FARO_TOKEN="${GRAFANA_OBSERVABILITY_FARO_TOKEN:-}" \
  -f "$WEB_DIR/Dockerfile.prod" \
  # ... rest of your build command ...
```

Replace `<app-id>` with the ID from your Grafana Cloud Frontend Observability app.

**When static assets look “stuck” (new image tag but old JS):** add **`--no-cache`** to `docker buildx build` for the website image so `npm run build` always runs fresh.

**Forward Grafana token from the host** (not automatic):

```bash
BUILD_ARGS=()
if [ -n "${GRAFANA_OBSERVABILITY_FARO_TOKEN:-}" ]; then
  BUILD_ARGS+=(--build-arg "GRAFANA_OBSERVABILITY_FARO_TOKEN=$GRAFANA_OBSERVABILITY_FARO_TOKEN")
fi
docker buildx build ... "${BUILD_ARGS[@]}" ...
```

---

## Alloy log pipeline for Nginx (observability-stack) {#alloy}

If using Grafana Alloy, split the `loki.process` block to handle Nginx JSON separately
from Java/Logstash JSON using `stage.match`:

```alloy
loki.process "<your-namespace>" {   // replace with your Kubernetes namespace name
  stage.drop {
    older_than          = "120h"
    drop_counter_reason = "timestamp_exceeds_cloud_ingest_window"
  }

  // Backend services (Java/Logstash JSON with mdc.traceId)
  stage.match {
    selector = `{service!="<frontend-app-name>"}`
    stage.json {
      expressions = {
        level    = "level",
        trace_id = "mdc.traceId",
        span_id  = "mdc.spanId",
      }
    }
    stage.labels { values = { level = "level" } }
    stage.structured_metadata { values = { traceId = "trace_id", spanId = "span_id" } }
    stage.drop { expression = ".*GET /health.*", drop_counter_reason = "healthcheck_probe" }
  }

  // Frontend Nginx JSON access logs
  stage.match {
    selector = `{service="<frontend-app-name>"}`
    stage.json {
      expressions = {
        status   = "status",
        method   = "method",
        uri      = "uri",
        duration = "duration",
      }
    }
    stage.labels { values = { status = "status" } }
    stage.structured_metadata { values = { method = "method", uri = "uri", duration = "duration" } }
    stage.drop { expression = ".*/health.*", drop_counter_reason = "healthcheck_probe" }
  }

  forward_to = [loki.write.grafana_cloud.receiver]
}
```

---

## .env.example additions

```bash
# Grafana Faro RUM — leave empty to disable in local dev.
# Get from Grafana Cloud → Frontend Observability → your app → Instrumentation.
VITE_FARO_URL=

# App version baked into Faro events. In CI: use git SHA or release tag.
VITE_APP_VERSION=dev

# Source map upload token — build-time only, never in the browser.
# Create at Grafana Cloud → Access Policies → new policy with sourcemaps:read + sourcemaps:write.
GRAFANA_OBSERVABILITY_FARO_TOKEN=
```
