# Java Development Rules for AI Agents

A shared rule set that Claude Code (and other CLAUDE.md-aware agents) load automatically when working on our Java projects. It encodes our standard stack — **Java 21, Spring Boot 3.x, Lombok, PostgreSQL on AWS RDS** — and the persistence guidance of [Vlad Mihalcea's High-Performance Java Persistence](https://vladmihalcea.com/books/high-performance-java-persistence/): simple by default, efficient by design.

The entry point is [`CLAUDE.md`](CLAUDE.md), which lists the non-negotiables and imports the detailed rules:

| File | Covers |
|---|---|
| [`rules/java-core.md`](rules/java-core.md) | Java 21 features, records, immutability, time/money types, error handling |
| [`rules/spring-boot.md`](rules/spring-boot.md) | Layering, constructor injection, web/service layer rules, application.yml conventions, testing |
| [`rules/lombok.md`](rules/lombok.md) | Allowed annotations, the entity bans (`@Data` etc.), lombok.config |
| [`rules/jpa-hibernate.md`](rules/jpa-hibernate.md) | Fetching, DTO projections, identifiers, equals/hashCode, associations, batching, locking (the Mihalcea core) |
| [`rules/postgresql-aws.md`](rules/postgresql-aws.md) | PostgreSQL usage, HikariCP sizing, RDS/Aurora, local Docker setup |
| [`rules/flyway.md`](rules/flyway.md) | Migration naming, immutability, zero-downtime schema changes |
| [`rules/observability.md`](rules/observability.md) | Actuator, Micrometer/Prometheus, Grafana dashboards, logging |

## Using the rules

### Option 1 — New project in this workspace (zero setup)

Claude Code loads `CLAUDE.md` from parent directories automatically. Create the project as a subdirectory and the rules just apply:

```bash
cd ~/Projects/java-dev
mkdir order-service && cd order-service
git init            # each project is its own repo; the workspace ignores it
claude
```

Then prompt, for example:

> Create a new Spring Boot service named `order-service` for managing customer orders, following the workspace rules end to end: Order/OrderItem entities, REST endpoints with request/response DTOs, Flyway baseline migration, compose.yaml for local PostgreSQL, Actuator + Prometheus configuration, and Testcontainers integration tests.

### Option 2 — Existing project elsewhere on your machine

Add one import line to the project's own `CLAUDE.md` (create the file if it doesn't exist):

```markdown
@~/Projects/java-dev/CLAUDE.md
```

Project-specific instructions below that line take precedence where they conflict. Note this only works on machines that have this repo checked out at that path — use Option 3 for anything teammates or CI touch.

### Option 3 — Team-shared repositories

Give every clone of the project identical rules, either by:

- **Copy** (simplest): copy `CLAUDE.md` and `rules/` into the project repo. Re-copy when the rules change.
- **Submodule** (stays in sync):

  ```bash
  git submodule add https://github.com/BroShow/java-dev.git .agent-rules
  echo "@.agent-rules/CLAUDE.md" >> CLAUDE.md
  ```

  Consumers update with `git submodule update --remote`.

## Sample prompts

**Scaffold a feature** (new or existing project):

> Add a `POST /orders/{id}/refunds` endpoint following the workspace rules end to end: Flyway migration, entity changes, DTOs, service with proper transaction boundaries, controller, and tests.

**Audit an existing codebase** — run this before letting agents make changes, so fixes are prioritized and deliberate:

> Audit this codebase against `rules/jpa-hibernate.md` and `rules/lombok.md`. Report every violation — EAGER associations, `@Data`/`@EqualsAndHashCode` on entities, IDENTITY id generation, missing pagination, open-in-view enabled, missing `@Version`, load-modify-save loops — as a prioritized list with file references. Do not change any code yet.

**Fix one audit finding at a time:**

> Fix finding #3 from the audit: convert the EAGER associations in the `invoice` package to LAZY and add `JOIN FETCH`/`@EntityGraph` to the queries that actually need those associations. Run the tests.

**Performance investigation:**

> The `/api/reports` endpoint is slow. Following `rules/jpa-hibernate.md`, check for N+1 queries, missing DTO projections, and missing pagination on this path. Show me the SQL Hibernate generates before and after your fix.

**Review a diff:**

> Review this branch's diff against all workspace rules and list violations before I open a PR.

## Maintaining the rules

- Rules are code: edit, commit, push. Copy-based consumers re-copy; submodule consumers `git submodule update --remote`.
- When adding a rule, prefer a *checkable* statement ("every FK column gets an index") over philosophy, and put it in the file where an agent would look for it.
- If a rule changes a non-negotiable, update the numbered list in `CLAUDE.md` too.
