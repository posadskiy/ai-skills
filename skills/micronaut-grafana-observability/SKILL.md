---
name: micronaut-grafana-observability
description: >-
  Implements backend observability (metrics, distributed tracing, structured
  logs, trace-log correlation) for Micronaut services using Micrometer/Prometheus,
  OpenTelemetry, Logstash JSON, and Grafana Alloy shipping to Grafana Cloud
  (Mimir, Tempo, Loki). Use when adding observability to a Micronaut service,
  wiring OTel traces, setting up Prometheus scraping, configuring structured
  JSON logs, or connecting a Java microservice to Grafana Cloud.
compatibility: Requires Micronaut 4+, Maven, Java 17+, Kubernetes deployment, and a Grafana Cloud account with Alloy deployed in-cluster.
metadata:
  author: posadskiy
  stack: Micronaut 4, Micrometer, Prometheus, OpenTelemetry OTLP, Logstash Logback Encoder, Grafana Alloy, Grafana Cloud (Mimir, Tempo, Loki)
---

# Micronaut + Grafana Cloud Observability

Three observability pillars wired together, all flowing through Grafana Alloy in-cluster to Grafana Cloud.

**See [references/code-templates.md](references/code-templates.md) for complete copy-paste configs.**

---

## What gets implemented

| Pillar | How |
|--------|-----|
| **Metrics** | Micrometer + Prometheus registry → `/prometheus` endpoint → Alloy pod-annotation scrape → Grafana Cloud Mimir |
| **Distributed traces** | `micronaut-tracing-opentelemetry-http` auto-instruments HTTP + JDBC → OTLP gRPC → Alloy → Grafana Cloud Tempo |
| **Structured logs** | `logstash-logback-encoder` (JSON to stdout) → Alloy `loki.source.kubernetes` → Grafana Cloud Loki |
| **Trace-log correlation** | OTel auto-injects `traceId`/`spanId` into SLF4J MDC → every JSON log line carries them; Alloy promotes them to Loki structured metadata |

---

## Step 1 — Maven dependencies

Add to the service (or shared parent) `pom.xml`.
Full XML → [references/code-templates.md](references/code-templates.md#maven-deps).

```xml
<!-- Micrometer + Prometheus: exposes /prometheus endpoint -->
<dependency>
  <groupId>io.micronaut.micrometer</groupId>
  <artifactId>micronaut-micrometer-core</artifactId>
</dependency>
<dependency>
  <groupId>io.micronaut.micrometer</groupId>
  <artifactId>micronaut-micrometer-registry-prometheus</artifactId>
</dependency>

<!-- OTel HTTP instrumentation: server/client HTTP + JDBC + MDC trace injection -->
<dependency>
  <groupId>io.micronaut.tracing</groupId>
  <artifactId>micronaut-tracing-opentelemetry-http</artifactId>
</dependency>
<dependency>
  <groupId>io.opentelemetry</groupId>
  <artifactId>opentelemetry-exporter-otlp</artifactId>
  <scope>compile</scope>
</dependency>

<!-- Structured JSON logging (Logstash format for Alloy/Loki) -->
<dependency>
  <groupId>net.logstash.logback</groupId>
  <artifactId>logstash-logback-encoder</artifactId>
  <version>7.4</version>
</dependency>
```

`micronaut-tracing-opentelemetry-http` auto-injects `traceId`/`spanId` into SLF4J MDC — no extra code needed for log correlation.

---

## Step 2 — Logback JSON config

Create `src/main/resources/logback.xml`.
Full file → [references/code-templates.md](references/code-templates.md#logback).

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
  </appender>
  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>
</configuration>
```

`LogstashEncoder` writes every log line as a JSON object to stdout. Alloy's `loki.source.kubernetes` picks it up and the `stage.json` pipeline extracts `level`, `mdc.traceId`, `mdc.spanId`.

---

## Step 3 — application.yml (metrics + tracing config)

Full file → [references/code-templates.md](references/code-templates.md#application-yml).

Key sections to add:

```yaml
micronaut:
  metrics:
    enabled: true
    export:
      prometheus:
        enabled: true
        descriptions: true
  tracing:
    opentelemetry:
      enabled: true

endpoints:
  prometheus:
    enabled: true
    sensitive: false   # Alloy scrapes without auth

otel:
  exporter:
    otlp:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4317}
      protocol: grpc
  traces:
    exporter: otlp    # Required: Micronaut default is "none" — without this, no spans are exported
    sampler:
      arg: ${OTEL_TRACES_SAMPLER_ARG:1.0}

logging:
  level:
    root: ${LOG_LEVEL_ROOT:INFO}
```

For `application-prod.yml`, you can override the sampler arg. Default to `1.0` (100%) unless
Grafana Cloud trace ingestion limits become a concern (high-traffic services):

```yaml
otel:
  traces:
    sampler:
      arg: "1.0"   # 100% by default; reduce (e.g. "0.2") only for high-traffic services
```

---

## Step 4 — Kubernetes Deployment

Add pod annotations so Alloy discovers the `/prometheus` endpoint, and set OTEL env vars for the trace pipeline.
Full YAML → [references/code-templates.md](references/code-templates.md#k8s-deployment).

**Pod annotations** (under `spec.template.metadata.annotations`):

```yaml
annotations:
  prometheus.io/scrape: "true"
  prometheus.io/path: "/prometheus"
  prometheus.io/port: "<service-port>"
```

**Container env vars**:

```yaml
- name: OTEL_SERVICE_NAME
  value: "my-service-name"
- name: OTEL_EXPORTER_OTLP_ENDPOINT
  value: "http://grafana-alloy.observability.svc.cluster.local:4317"
- name: OTEL_TRACES_SAMPLER
  value: parentbased_traceidratio
- name: OTEL_TRACES_SAMPLER_ARG
          value: "1.0"   # reduce to e.g. "0.2" only for high-traffic services
```

The Alloy service DNS (`grafana-alloy.observability.svc.cluster.local:4317`) assumes Alloy is in the `observability` namespace. Adjust if different.

---

## Step 5 — Grafana Alloy config

The Alloy config handles all three pillars. It must be deployed in-cluster (e.g. via Helm `values.yaml` or a ConfigMap).
Full config → [references/code-templates.md](references/code-templates.md#alloy-config).

### Metrics pipeline
```alloy
discovery.relabel "costy_metrics" {
  // keep pods with prometheus.io/scrape=true annotation
  // extract prometheus.io/path and prometheus.io/port annotations
}
prometheus.scrape "costy" {
  targets         = discovery.relabel.costy_metrics.output
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
  scrape_interval = "60s"
}
prometheus.remote_write "grafana_cloud" {
  endpoint { url = "https://prometheus-prod-<region>.grafana.net/api/prom/push" }
}
```

### Logs pipeline
```alloy
loki.source.kubernetes "costy" { ... }
loki.process "costy" {
  stage.match { selector = `{service!="frontend"}` }
    stage.json { expressions = { level="level", trace_id="mdc.traceId", span_id="mdc.spanId" } }
    stage.labels { values = { level = "level" } }
    stage.structured_metadata { values = { traceId = "trace_id", spanId = "span_id" } }
    stage.drop { expression = ".*GET /health.*" }  // filter healthcheck noise
  }
}
loki.write "grafana_cloud" { endpoint { url = "https://logs-prod-<region>.grafana.net/loki/api/v1/push" } }
```

### Traces pipeline
```alloy
otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output { traces = [otelcol.processor.batch.default.input] }
}
otelcol.processor.batch "default" { timeout = "10s" ... }
otelcol.exporter.otlphttp "grafana_cloud" {
  client { endpoint = "https://otlp-gateway-prod-<region>.grafana.net/otlp" }
}
```

### Required env vars in Alloy pod:
| Variable | Used by |
|----------|---------|
| `GRAFANA_OBSERVABILITY_USER_TOKEN` | Loki write + Mimir remote_write basic_auth |
| `GRAFANA_OBSERVABILITY_OTLP_TOKEN` | OTLP exporter basic_auth |
| `GRAFANA_CLOUD_OTLP_USER` | OTLP exporter username (OpenTelemetry Instance ID) |

---

## Step 6 — Grafana Cloud setup

1. **Mimir (metrics)**: note the remote_write URL from Grafana Cloud → Connections → Hosted Prometheus → `Remote Write Endpoint`. Username is a numeric ID (e.g. `3097590`). Use an Access Policy token with `metrics:write`.
2. **Loki (logs)**: note the Loki push URL → Connections → Hosted Logs. Username is a numeric ID. Same token can cover both if scoped with `logs:write`.
3. **Tempo (traces)**: note the OTLP gateway URL → Connections → Hosted Traces → OpenTelemetry. Username = OpenTelemetry Instance ID (numeric). Use a token with `traces:write`.

---

## Where to see results

| What | Where in Grafana Cloud |
|------|----------------------|
| JVM metrics, HTTP rate/latency/errors | Explore → Metrics → filter by `service="<app-name>"` |
| Structured logs with log level | Explore → Loki → `{service="<app-name>"}` |
| Error logs only | Loki → `{service="<app-name>", level="ERROR"}` |
| Distributed traces | Explore → Tempo → search by `service.name` |
| Log → trace correlation | Click `traceId` in a Loki log line → jumps to Tempo span |
| Trace → log correlation | In Tempo span, click "Logs for this span" |
