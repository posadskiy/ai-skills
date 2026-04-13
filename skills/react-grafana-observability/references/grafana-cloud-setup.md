# Grafana Cloud Setup Guide

Step-by-step UI walkthrough for everything that needs to be configured in Grafana Cloud.

---

## 1. Create a Frontend Observability Application

This gives you the Faro collector URL (`VITE_FARO_URL`) and the `appId` / `stackId` values
for the source map uploader.

**Each app has its own ingest path.** The segment after `/collect/` is a **per-app routing key** (sometimes shown as a hex string). It is **not** interchangeable with another Grafana Frontend app — copying another project’s URL sends RUM to that app and often causes **CORS** errors for your domain. Example shape (illustration only):

`https://faro-collector-prod-eu-west-2.grafana.net/collect/451ee59b19fd23cfa38502c8993ecf6a`

1. Sign in to [grafana.com](https://grafana.com) → select your organization → open your stack → **Launch** Grafana
2. Left menu → expand **Observability** → select **Frontend**
3. Click **Create new** → fill in:
   - **App Name**: e.g. `my-project` (use this exact name as `appName` in `vite.config.ts` and `faro.ts`)
   - **CORS Allowed Origins**: your production domain (e.g. `https://my-project.com`)
   - Default attributes: leave empty
4. Click **Create**
5. On the **Web SDK Configuration** page Grafana shows a pre-filled `initializeFaro` snippet.
   From this snippet, copy:
   - `url` value → this is your **`VITE_FARO_URL`**
   - `appId` value → use in `vite.config.ts` `faroUploader({ appId: ... })`
   - `stackId` value → use in `vite.config.ts` `faroUploader({ stackId: ... })`

> The collector URL looks like:
> `https://faro-collector-prod-eu-west-2.grafana.net/collect/<app-id>`

---

## 2. Create the Source Map Upload Token

This gives you **`GRAFANA_OBSERVABILITY_FARO_TOKEN`** for the `faroUploader` plugin.

### From inside your Grafana stack (recommended)

1. Left menu → **Administration** → **Users and access** → **Cloud access policies**
2. Click **Create access policy**
3. Display name: `faro-observability`
4. Scopes: find **sourcemaps** row → check **Read** ✅ and **Write** ✅
   (if sourcemaps is not visible, use the **Add scope** dropdown and type `sourcemaps`)
5. Click **Create**
6. Under the new policy → click **Add token**
7. Token name: e.g. `faro-observability-token`
8. Expiry: 1 year (recommended) or no expiry
9. Click **Generate** → **copy the `glc_...` token immediately** — shown only once

### From the Cloud Portal (alternative)

1. [grafana.com](https://grafana.com) → your organization → **Security** → **Access Policies**
2. Same steps from step 2 above, with Realm set to your specific stack

---

## 3. Enable Alerting (7 predefined rules)

After the app is receiving data (deploy first, then come back):

1. Grafana Cloud → **Frontend Observability** → your app → **Settings** → **Alerting**
2. Enable all 7 rules — they use Google's Core Web Vitals "poor" thresholds:

| Alert | Threshold | What it catches |
|-------|-----------|----------------|
| **Errors Count** | Day-over-day increase | Gradual error regression after a deploy |
| **New Errors** | Any new error fingerprint in 24h | New bugs introduced by a release |
| **Web Vitals — LCP** | > 4000ms | Page main content taking 4+ seconds |
| **Web Vitals — CLS** | > 0.25 | Visible layout jumps during load |
| **Web Vitals — TTFB** | > 1800ms | Slow server / network response |
| **Web Vitals — FCP** | > 3000ms | Nothing visible for 3 seconds |
| **Web Vitals — INP** | > 500ms | Clicks/taps feel frozen (replaced FID in 2024) |

> Enable **all 7**. Thresholds can be tightened later from the app settings page.

---

## 4. Synthetic Monitoring (uptime checks)

No in-cluster agent needed for HTTP checks — Grafana Cloud runs probes from multiple regions.

1. Grafana Cloud → **Synthetic Monitoring** → **Add check** → **HTTP**
2. Create these checks:

| Check | URL | Expected response |
|-------|-----|-------------------|
| Frontend up | `https://<your-domain>/` | Status 200, body contains `<div id="root">` |
| Health endpoint | `https://<your-domain>/health` | Status 200, body `healthy` |
| API availability | `https://api.<your-domain>/health` | Status 200 |

3. For each check:
   - Enable **TLS certificate expiry** alert (default: warn at 30 days, critical at 7 days)
   - Set **alert on 2 consecutive failures** (~2 min detection time)
4. Alert contact point: configure in **Alerting** → **Contact points** (email, Slack, PagerDuty, etc.)

---

## 5. Where to see results

| What | Path in Grafana Cloud |
|------|-----------------------|
| RUM dashboard (Web Vitals, errors, sessions) | Frontend → Frontend Observability → your app |
| Session list with JS errors | Frontend Observability → Sessions |
| Per-route performance breakdown | Frontend Observability → Performance → filter by View |
| De-obfuscated stack traces | Frontend Observability → Errors (requires source map upload) |
| Nginx access logs | Explore → Loki → `{namespace="costy", service="<frontend-app>"}` |
| 5xx error filter | Loki → `{service="<frontend-app>"} \| status =~ "5.."` |
| Full browser→backend traces | Explore → Tempo → search by service name |
| Uptime / response time history | Synthetic Monitoring → your check → dashboard |
| Fired alerts | Alerting → Alert rules / Fired alerts |

---

## 6. Verify data is flowing

**After first deploy with Faro wired in:**

```
Grafana Cloud → Frontend Observability → your app
→ should see session count increase within 1-2 minutes of visiting the app
```

**Logs (Nginx):**
```
Explore → Loki → {namespace="costy"}
→ look for lines with "method", "status", "uri" fields
```

**Source maps:**
```
Frontend Observability → Errors → click any error
→ stack trace should show original file names and line numbers (not minified)
→ if still minified: check build output for "Faro: uploading source maps" log line
```
