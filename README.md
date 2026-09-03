# Java Development Rules for AI Agents

A shared rule set that Claude Code (and other CLAUDE.md-aware agents) load automatically when working on our Java projects. It encodes our standard stack — **Java 21, Spring Boot 3.x, Lombok, PostgreSQL on AWS RDS** — and guidance from three trusted sources: [Vlad Mihalcea's High-Performance Java Persistence](https://vladmihalcea.com/books/high-performance-java-persistence/) for the persistence layer, Joshua Bloch's *Effective Java* for core Java, and Oliver Drotbohm's modular-monolith approach ([Spring Modulith](https://spring.io/projects/spring-modulith)) for architecture: simple by default, efficient by design.

The entry point is [`CLAUDE.md`](CLAUDE.md), which lists the non-negotiables and imports the detailed rules:

| File | Covers |
|---|---|
| [`rules/java-core.md`](rules/java-core.md) | Java 21 features, records, immutability, time/money types, error handling, Effective Java (object contracts, API design, generics, lambdas/streams) |
| [`rules/spring-boot.md`](rules/spring-boot.md) | Layering, constructor injection, web/service layer rules, application.yml conventions, testing |
| [`rules/architecture.md`](rules/architecture.md) | Module boundaries (Spring Modulith), modulith-first stance, aggregate design |
| [`rules/lombok.md`](rules/lombok.md) | Allowed annotations, the entity bans (`@Data` etc.), lombok.config |
| [`rules/jpa-hibernate.md`](rules/jpa-hibernate.md) | Fetching, DTO projections, identifiers, equals/hashCode, associations, batching, locking (the Mihalcea core) |
| [`rules/postgresql-aws.md`](rules/postgresql-aws.md) | PostgreSQL usage, HikariCP sizing, RDS/Aurora, local Docker setup |
| [`rules/flyway.md`](rules/flyway.md) | Migration naming, immutability, zero-downtime schema changes |
| [`rules/observability.md`](rules/observability.md) | Actuator, Micrometer/Prometheus, Grafana dashboards, logging |
| [`rules/static-analysis.md`](rules/static-analysis.md) | Palantir Java Format via Spotless, SonarQube posture (Sonar way, blocking core), JaCoCo coverage |

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

### Using with Kiro

Kiro doesn't read `CLAUDE.md` and has no equivalent of the `@rules/...` import syntax — but its [steering files](https://kiro.dev/docs/steering/) (`.kiro/steering/*.md`) default to `inclusion: always` when they have no front matter, which matches how Claude Code loads these rules. The rule files work verbatim; only the entry point needs recreating:

```bash
mkdir -p .kiro/steering
cp ~/Projects/java-dev/rules/*.md .kiro/steering/
# Entry point: CLAUDE.md minus the @rules/ import lines (the files are loaded directly now)
grep -v '^@rules/' ~/Projects/java-dev/CLAUDE.md > .kiro/steering/00-java-non-negotiables.md
```

Re-copy when the rules change, as with the copy variant of Option 3. Two notes:

- Kiro supports scoping a steering file via front matter (`inclusion: fileMatch`), e.g. `flyway.md` to `**/db/migration/**`. Prefer the `always` default anyway: these are generation rules, and a match on existing files won't fire when the agent is creating the first migration.
- Kiro also loads [`AGENTS.md`](https://agents.md) files, but they don't follow `@` imports either — so they offer no advantage over steering files for this repo.

## Sample prompts

**Scaffold a feature** (new or existing project):

> Add a `POST /orders/{id}/refunds` endpoint following the workspace rules end to end: Flyway migration, entity changes, DTOs, service with proper transaction boundaries, controller, and tests.

**Building blocks** — one layer at a time, when you'd rather review each piece than take a whole feature in one diff:

*Add an entity (+ migration):*

> Add a `Refund` entity to the `order` module as part of the Order aggregate, per `rules/jpa-hibernate.md`, with its Flyway migration per `rules/flyway.md`.

*Add a repository:*

> Add a repository for the `Refund` aggregate root per `rules/jpa-hibernate.md`, including a paginated DTO projection for the refund list screen. Show me the SQL Hibernate generates for the projection query.

*Add a service:*

> Add a `RefundService` to the `order` module per `rules/spring-boot.md`, covering create, approve, and lookup-by-order.

*Add a controller:*

> Add a `RefundController` exposing `RefundService` per `rules/spring-boot.md`, with a `@WebMvcTest` covering validation failures and the happy path.

*Add a module:*

> Add a new `payment` module per `rules/architecture.md`. It will need order data — wire that dependency the way the rules require, and make sure the `ApplicationModules.verify()` test still passes.

*Add tests for existing code:*

> Add persistence tests for the `order` module per the testing rules in `rules/spring-boot.md` and `rules/postgresql-aws.md`, with statement-count assertions on the query paths most likely to regress into N+1.

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
