# Observability Rules (Actuator, Micrometer/Prometheus, Grafana)

Every service ships with metrics and health endpoints from its first commit — observability is not a later add-on.

## Spring Boot Actuator

- Every project includes `spring-boot-starter-actuator` and `micrometer-registry-prometheus`.
- Expose only what's needed:

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus
  endpoint:
    health:
      probes:
        enabled: true
      show-details: when-authorized
```

- **Actuator endpoints are never internet-facing.** Either serve them on a separate `management.server.port` scraped only inside the VPC, or block `/actuator/**` at the load balancer. Endpoints like `env`, `heapdump`, and `threaddump` leak secrets and stay disabled in production.
- Use the liveness/readiness probes (`/actuator/health/liveness`, `/actuator/health/readiness`) for ECS/Kubernetes health checks — not the bare `/actuator/health`, which can flap when an external dependency blips.
- `info` is populated from build metadata (`spring-boot-maven-plugin` `build-info` goal) so every running instance reports its version and commit.

## Metrics (Micrometer → Prometheus)

- Prometheus **scrapes** `/actuator/prometheus`; never push, except batch jobs that finish before a scrape (Pushgateway).
- Tag every metric with the service and environment once, globally:

```yaml
management:
  metrics:
    tags:
      application: ${spring.application.name}
      environment: ${ENVIRONMENT:local}
  observations:
    key-values:
      # applies the same tags to traces if tracing is enabled
```

- Enable latency histograms for HTTP so Grafana can compute real percentiles:

```yaml
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
```

- Custom business metrics use `MeterRegistry` (injected) or `@Timed`/`@Counted` on service methods. Naming: lowercase dot-separated, a noun plus unit implied by type — `orders.placed` (counter), `payment.processing` (timer).
- **Never use unbounded values as tags** (user IDs, order IDs, free text). Tag cardinality multiplies time series and will take down Prometheus. Tags are for low-cardinality dimensions: status, type, outcome.
- Counters for events, timers for durations, gauges only for current-state values you can read on demand (queue depth, pool size).

## What every service dashboard shows (Grafana)

Dashboards are **provisioned as code** (JSON in the repo or Terraform/Grafana provider) — never hand-edited only in the UI.

The standard per-service dashboard covers:

1. **RED**: request rate, error rate (5xx %), duration p50/p95/p99 — from `http_server_requests_seconds`.
2. **JVM**: heap used vs max, GC pause time, thread count — from the built-in `jvm_*` metrics.
3. **HikariCP**: `hikaricp_connections_active`, `hikaricp_connections_pending`, connection acquire time, and connection usage (lease) time. **Pending > 0 is the earliest visible symptom of the persistence problems in `jpa-hibernate.md`** (N+1 storms, long transactions, missing pagination) — alert on it. Pool sizing follows Mihalcea's FlexyPool philosophy: size from *measured* concurrent-usage and acquire-time histograms, never from guesses — Hikari's built-in Micrometer metrics give you the same data FlexyPool was built to collect.
4. **Datasource/Flyway**: migration state, connection creation rate.

Baseline alerts per service: sustained 5xx rate, p99 latency above SLO, Hikari pending connections, heap > 90% after full GC. Alert on symptoms users feel, not on every metric.

## Logging (minimum bar)

- JSON-structured logs in deployed environments (`logging.structured.format.console=ecs` on Boot 3.4+), human-readable locally.
- Include trace/correlation IDs via Micrometer Tracing when services call each other.
- Log at `INFO` for state changes, `WARN` for handled anomalies, `ERROR` only when something needs human attention. Never log secrets, tokens, or full request bodies.
