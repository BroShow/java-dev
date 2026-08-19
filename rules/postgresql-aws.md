# PostgreSQL & AWS RDS Rules

PostgreSQL is the default database for all projects, typically running on AWS RDS (or Aurora PostgreSQL). These rules follow Vlad Mihalcea's PostgreSQL-specific guidance plus RDS operational practice.

## PostgreSQL usage

- **Play to the database's strengths.** Native SQL queries are encouraged when JPQL falls short: window functions, CTEs, `LATERAL` joins, `INSERT ... ON CONFLICT` (upserts). Don't reimplement set-based logic in Java loops.
- **JSONB for schema-less data.** Map with Hibernate 6's built-in support — no custom types needed:

```java
@JdbcTypeCode(SqlTypes.JSON)
@Column(columnDefinition = "jsonb")
private Map<String, Object> attributes;
```

  Use `jsonb` (indexed, binary), never plain `json`. Add a GIN index in the Flyway migration if the column is queried.
- **Timestamps are `timestamptz`** in DDL, mapped to `Instant` (or `OffsetDateTime`) in entities. Never `timestamp without time zone`.
- Prefer `text` over `varchar(n)` unless a length limit is a genuine business rule — they perform identically in PostgreSQL.
- **Pick the narrowest type that fits the data, and prefer the native type over a generic one.** Compact rows mean more of the working set and more index entries fit in the buffer pool — the cheapest performance win there is (Mihalcea). Use `int` over `bigint` for counts and codes that will never exceed 2 billion, `smallint` for enum ordinals (see `jpa-hibernate.md`), `inet`/`cidr` for IP addresses, `uuid` for UUIDs (never `text`), `numeric(p,s)` with an explicit scale for money. Primary keys stay `bigint` regardless — see `jpa-hibernate.md`.
- IDs come from explicit sequences, never `SERIAL`/`BIGSERIAL` (see `jpa-hibernate.md`).
- For coordination beyond row locks (e.g. singleton jobs across instances), use **advisory locks** (`pg_advisory_xact_lock`) — or ShedLock, which uses them — instead of home-grown lock tables.
- Rely on MVCC: readers don't block writers and vice versa. Don't add `LockModeType.PESSIMISTIC_READ` "just in case" — lock only when the use case proves it.

## JDBC driver & connection pool

- Standard JDBC URL:

```
jdbc:postgresql://<host>:5432/<db>?reWriteBatchedInserts=true&sslmode=require
```

- **HikariCP stays small.** RDS `max_connections` is limited by instance memory, and PostgreSQL degrades with too many connections. Default:

```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 10
      minimum-idle: 10
      max-lifetime: 900000        # 15 min — below any NAT/LB idle timeout
      connection-timeout: 5000
```

  Size the pool from `(cores × 2) + effective_spindle_count` per instance, not from expected user count. More connections is almost never the fix.
- **Leave server-side prepared statements on.** After `prepareThreshold` executions (default 5) the driver promotes a statement to a named server-side prepared statement, so PostgreSQL reuses the parse and plan instead of re-parsing every call — Mihalcea's statement-caching tip. The driver's own cache (`preparedStatementCacheQueries=256`, `preparedStatementCacheSizeMiB=5`) is sized sensibly by default; raise it only for an application with many distinct statements, and never set `prepareThreshold=0` to "fix" an error without understanding the cause.
  - **Transaction-level connection pooling breaks this.** Behind **RDS Proxy**, prepared statements pin the session to a backend connection and defeat multiplexing; behind **PgBouncer in transaction mode** (pre-1.21) they fail outright. If either sits in front of the database, either set `prepareThreshold=0` deliberately and record why, or use a proxy version that supports protocol-level prepared statements. Pick one — don't discover it in production.
  - Hibernate's `query.in_clause_parameter_padding` (see `jpa-hibernate.md`) exists to make this cache actually hit for `IN` queries.
- Set a `statement_timeout` (via `spring.datasource.hikari.data-source-properties.options` or the RDS parameter group) so a runaway query can't hold connections forever.

## AWS RDS

- **TLS always**: `sslmode=require` minimum; `verify-full` with the AWS RDS CA bundle for anything handling sensitive data.
- **Credentials from AWS Secrets Manager** (with rotation) or **IAM database authentication** — never in `application.yml`, never in environment files committed to git. Use `spring-cloud-aws` starters for Secrets Manager integration.
- **Read replicas**: route `@Transactional(readOnly = true)` traffic to the reader endpoint when read scaling is needed (e.g. via `LazyConnectionDataSourceProxy` + a routing `DataSource`, or the AWS Advanced JDBC Wrapper for Aurora). This is why every read path must already be marked `readOnly = true`.
- **Aurora / failover**: prefer the [AWS Advanced JDBC Wrapper](https://github.com/aws/aws-advanced-jdbc-wrapper) around the PostgreSQL driver for fast failover detection instead of waiting on DNS TTLs.
- **RDS Proxy** when connection counts spike (Lambda consumers, many small services sharing one database) — don't crank `max_connections` in the parameter group.
- Database tuning belongs in the **RDS parameter group** (IaC-managed), not in application code. Trust RDS defaults for `shared_buffers` etc. unless Performance Insights shows a problem.
- **Performance Insights on, always.** When a query is slow, read its execution plan (`EXPLAIN (ANALYZE, BUFFERS)`) before touching Hibernate settings.
- Schema changes run as Flyway migrations against RDS from CI/CD — never by hand in a console. Avoid long `ACCESS EXCLUSIVE` locks: create indexes `CONCURRENTLY`, add columns without volatile defaults on hot tables.

## Local development (Docker)

- Local PostgreSQL runs in **Docker**, pinned to the same major version as production RDS. Every project ships a `compose.yaml` at the repo root:

```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: app
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:
```

- Add the `spring-boot-docker-compose` dependency (scope `runtime`, dev only): Spring Boot then starts the compose services and wires the datasource automatically on `bootRun` — no hand-maintained local credentials in `application.yml`.
- Local containers may use trivial credentials like the above; they must be obviously non-production values. `sslmode=require` applies to RDS, not local Docker.
- Flyway runs the same migrations locally as in RDS — never a separate "dev schema" path. Resetting local state is `docker compose down -v`, not manual SQL.

## Testing

- Integration tests run against real PostgreSQL via **Testcontainers**, pinned to the same major version as production RDS (e.g. `postgres:16`). Never H2 — it lies about dialect, locking, and JSON behavior.
- Use `tmpfs` mounts for the Testcontainers data directory to keep test suites fast.

## References

- Vlad Mihalcea — 9 PostgreSQL high-performance tips: https://vladmihalcea.com/9-postgresql-high-performance-performance-tips/
