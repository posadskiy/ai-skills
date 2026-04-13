# Code Templates

Complete copy-paste configs for Micronaut + Grafana Cloud observability.

---

## Maven dependencies {#maven-deps}

Add to the parent or service `pom.xml`. The `logstash-logback-encoder` version must be declared
in `<dependencyManagement>` in the parent POM (or specified inline). Micrometer and OTel versions
are managed by `io.micronaut.platform:micronaut-platform`.

```xml
<properties>
  <logstash-logback-encoder.version>7.4</logstash-logback-encoder.version>
</properties>

<dependencyManagement>
  <dependencies>
    <dependency>
      <groupId>io.micronaut.platform</groupId>
      <artifactId>micronaut-platform</artifactId>
      <version>${micronaut.version}</version>
      <type>pom</type>
      <scope>import</scope>
    </dependency>
    <dependency>
      <groupId>net.logstash.logback</groupId>
      <artifactId>logstash-logback-encoder</artifactId>
      <version>${logstash-logback-encoder.version}</version>
    </dependency>
  </dependencies>
</dependencyManagement>

<dependencies>
  <!-- Micrometer + Prometheus: exposes /prometheus endpoint for Alloy to scrape -->
  <dependency>
    <groupId>io.micronaut.micrometer</groupId>
    <artifactId>micronaut-micrometer-core</artifactId>
  </dependency>
  <dependency>
    <groupId>io.micronaut.micrometer</groupId>
    <artifactId>micronaut-micrometer-registry-prometheus</artifactId>
  </dependency>

  <!--
    OTel HTTP instrumentation: auto-instruments server/client HTTP + JDBC,
    and injects traceId/spanId into SLF4J MDC so LogstashEncoder
    includes them in every JSON log line automatically.
  -->
  <dependency>
    <groupId>io.micronaut.tracing</groupId>
    <artifactId>micronaut-tracing-opentelemetry-http</artifactId>
  </dependency>
  <dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
    <scope>compile</scope>
  </dependency>

  <!-- Structured JSON logging -->
  <dependency>
    <groupId>net.logstash.logback</groupId>
    <artifactId>logstash-logback-encoder</artifactId>
  </dependency>
</dependencies>
```

---

## logback.xml {#logback}

`src/main/resources/logback.xml`

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

`LogstashEncoder` writes each log event as a JSON object including `@timestamp`, `level`, `logger_name`,
`message`, and the full MDC map. With OTel instrumentation active, MDC automatically contains
`traceId` and `spanId` for any log emitted inside a traced request.

---

## application.yml {#application-yml}

`src/main/resources/application.yml` — add or merge the following sections:

```yaml
micronaut:
  application:
    name: my-service-name   # used as service.name in OTel resource attributes
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
    # Required: Micronaut default is "none" — without this, no spans are exported to Alloy/Tempo.
    exporter: otlp
    sampler:
      arg: ${OTEL_TRACES_SAMPLER_ARG:1.0}   # 100% in local dev

logging:
  level:
    root: ${LOG_LEVEL_ROOT:INFO}
    com.example.myservice: ${LOG_LEVEL_APP:INFO}
    io.micronaut: ${LOG_LEVEL_MICRONAUT:INFO}
```

`src/main/resources/application-prod.yml` — production overrides:

```yaml
otel:
  traces:
    sampler:
      arg: "1.0"   # 100% by default; reduce to e.g. "0.2" only for high-traffic services
```

---

## Kubernetes Deployment {#k8s-deployment}

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service
  namespace: costy
  labels:
    app: my-service
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-service
  template:
    metadata:
      labels:
        app: my-service
      annotations:
        # Alloy uses these annotations to auto-discover the Prometheus endpoint
        prometheus.io/scrape: "true"
        prometheus.io/path: "/prometheus"
        prometheus.io/port: "<service-port>"
    spec:
      containers:
      - name: my-service
        image: myrepo/my-service:latest
        ports:
        - containerPort: <service-port>
        env:
        - name: MICRONAUT_ENVIRONMENTS
          value: "prod"
        # OTel trace pipeline config
        - name: OTEL_SERVICE_NAME
          value: "my-service"
        - name: OTEL_EXPORTER_OTLP_ENDPOINT
          value: "http://grafana-alloy.observability.svc.cluster.local:4317"
        - name: OTEL_TRACES_SAMPLER
          value: parentbased_traceidratio
        - name: OTEL_TRACES_SAMPLER_ARG
          value: "1.0"   # reduce to e.g. "0.2" only for high-traffic services
        # Health probes (Micronaut exposes /health by default)
        livenessProbe:
          httpGet:
            path: /health
            port: <service-port>
          initialDelaySeconds: 60
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health
            port: <service-port>
          initialDelaySeconds: 30
          periodSeconds: 10
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
      restartPolicy: Always
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
  namespace: costy
spec:
  type: ClusterIP
  ports:
  - port: <service-port>
    targetPort: <service-port>
    protocol: TCP
  selector:
    app: my-service
```

---

## Grafana Alloy config {#alloy-config}

`config.alloy` — deploy as a ConfigMap or embed in Helm `values.yaml`.

Required env vars in the Alloy pod:
- `GRAFANA_OBSERVABILITY_USER_TOKEN` — Access Policy token with `metrics:write` + `logs:write`
- `GRAFANA_OBSERVABILITY_OTLP_TOKEN` — Access Policy token with `traces:write`
- `GRAFANA_CLOUD_OTLP_USER` — OpenTelemetry Instance ID (numeric, from Grafana Cloud → Connections → Hosted Traces)

Replace `<region>`, `<mimir-username>`, `<loki-username>` with values from your Grafana Cloud stack.

```alloy
logging {
  level  = "info"
  format = "logfmt"
}

// ── Metrics pipeline ────────────────────────────────────────────────────────

discovery.kubernetes "costy_pods" {
  role = "pod"
  namespaces { names = ["costy"] }
}

discovery.relabel "costy_metrics" {
  targets = discovery.kubernetes.costy_pods.targets

  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_scrape"]
    action        = "keep"
    regex         = "true"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_annotation_prometheus_io_path"]
    action        = "replace"
    target_label  = "__metrics_path__"
    regex         = "(.+)"
  }
  rule {
    source_labels = ["__address__", "__meta_kubernetes_pod_annotation_prometheus_io_port"]
    action        = "replace"
    regex         = "([^:]+)(?:\\d+)?;(\\d+)"
    replacement   = "$1:$2"
    target_label  = "__address__"
  }
  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "service"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
}

prometheus.scrape "costy" {
  targets         = discovery.relabel.costy_metrics.output
  forward_to      = [prometheus.remote_write.grafana_cloud.receiver]
  scrape_interval = "60s"
  scrape_timeout  = "8s"
}

prometheus.remote_write "grafana_cloud" {
  endpoint {
    url = "https://prometheus-prod-<region>.grafana.net/api/prom/push"
    basic_auth {
      username = "<mimir-username>"
      password = sys.env("GRAFANA_OBSERVABILITY_USER_TOKEN")
    }
  }
  external_labels = {
    cluster = "k3s-prod",
    env     = "production",
  }
}

// ── Logs pipeline ───────────────────────────────────────────────────────────

discovery.kubernetes "costy_pods_logs" {
  role = "pod"
  namespaces { names = ["costy"] }
}

discovery.relabel "costy_logs" {
  targets = discovery.kubernetes.costy_pods_logs.targets

  rule {
    source_labels = ["__meta_kubernetes_namespace"]
    target_label  = "namespace"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_name"]
    target_label  = "pod"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_label_app"]
    target_label  = "service"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_container_name"]
    target_label  = "container"
  }
  rule {
    source_labels = ["__meta_kubernetes_pod_node_name"]
    target_label  = "node"
  }
}

loki.source.kubernetes "costy" {
  targets    = discovery.relabel.costy_logs.output
  forward_to = [loki.process.costy.receiver]
}

loki.process "costy" {
  // Drop log lines older than 5 days; Grafana Cloud rejects lines past ~7d.
  stage.drop {
    older_than          = "120h"
    drop_counter_reason = "timestamp_exceeds_cloud_ingest_window"
  }

  // Backend services: parse Logstash JSON, extract level + trace correlation fields.
  stage.match {
    selector = `{service!="<frontend-service-name>"}`

    stage.json {
      expressions = {
        level    = "level",
        trace_id = "mdc.traceId",
        span_id  = "mdc.spanId",
      }
    }
    stage.labels {
      values = { level = "level" }
    }
    stage.structured_metadata {
      values = {
        traceId = "trace_id",
        spanId  = "span_id",
      }
    }
    // Drop noisy healthcheck probes from log streams
    stage.drop {
      expression          = ".*GET /health.*"
      drop_counter_reason = "healthcheck_probe"
    }
  }

  forward_to = [loki.write.grafana_cloud.receiver]
}

loki.write "grafana_cloud" {
  endpoint {
    url = "https://logs-prod-<region>.grafana.net/loki/api/v1/push"
    basic_auth {
      username = "<loki-username>"
      password = sys.env("GRAFANA_OBSERVABILITY_USER_TOKEN")
    }
  }
  external_labels = {
    cluster = "k3s-prod",
    env     = "production",
  }
}

// ── Traces pipeline ─────────────────────────────────────────────────────────

otelcol.receiver.otlp "default" {
  grpc { endpoint = "0.0.0.0:4317" }
  http { endpoint = "0.0.0.0:4318" }
  output { traces = [otelcol.processor.batch.default.input] }
}

otelcol.processor.batch "default" {
  timeout             = "10s"
  send_batch_size     = 256
  send_batch_max_size = 512
  output { traces = [otelcol.exporter.otlphttp.grafana_cloud.input] }
}

otelcol.auth.basic "grafana_cloud" {
  username = sys.env("GRAFANA_CLOUD_OTLP_USER")
  password = sys.env("GRAFANA_OBSERVABILITY_OTLP_TOKEN")
}

otelcol.exporter.otlphttp "grafana_cloud" {
  client {
    endpoint = "https://otlp-gateway-prod-<region>.grafana.net/otlp"
    auth     = otelcol.auth.basic.grafana_cloud.handler
  }
}
```

---

## Grafana Cloud credentials lookup

| Credential | Where to find it |
|------------|-----------------|
| Mimir remote_write URL + username | Grafana Cloud → Connections → Hosted Prometheus metrics → Remote Write Endpoint |
| Loki push URL + username | Grafana Cloud → Connections → Hosted Logs → Push logs using the Loki HTTP API |
| OTLP gateway URL + `GRAFANA_CLOUD_OTLP_USER` | Grafana Cloud → Connections → Hosted Traces → OpenTelemetry |
| Access Policy token | Grafana Cloud → Security → Access Policies → Create policy → choose scopes → Generate token |

Recommended Access Policy scopes per token:

| Token | Scopes |
|-------|--------|
| `GRAFANA_OBSERVABILITY_USER_TOKEN` | `metrics:write`, `logs:write` |
| `GRAFANA_OBSERVABILITY_OTLP_TOKEN` | `traces:write` |
