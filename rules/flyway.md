# Flyway Rules

Vlad Mihalcea's position ([Flyway Database Schema Migrations](https://vladmihalcea.com/flyway-database-schema-migrations/)): never run migration scripts manually — manual actions are prone to human error; an automated tool applies them consistently, and because the same scripts run in QA first, production deployments are validated before they happen. That is the model here.

Flyway owns the schema. Hibernate never creates or alters tables: `spring.jpa.hibernate.ddl-auto=validate` everywhere (`validate` confirms mappings match the migrated schema at startup).

## Layout & naming

- Migrations live in `src/main/resources/db/migration`.
- Versioned migrations: `V<version>__<snake_case_description>.sql` — e.g. `V7__add_order_status_index.sql`. Use plain incrementing integers (`V1`, `V2`, …); timestamps only if many developers collide on numbers.
- Repeatable migrations (`R__create_views.sql`) only for idempotent objects: views, functions, procedures. Never for tables or data.
- One logical change per migration. Small migrations are reviewable and individually revertible-by-forward-fix.

## Immutability

- **Never edit a migration that has run anywhere beyond your machine.** Flyway checksums will fail. Fix mistakes with a new forward migration.
- Never use `flyway.repair` or `out-of-order` in a deployed environment to paper over process failures — fix the process.

## Writing migrations (PostgreSQL)

- DDL follows the entity rules: `bigint` IDs with explicit sequences (matching `@SequenceGenerator`, `INCREMENT BY 50`), `timestamptz` timestamps, `text` columns, `jsonb` with GIN indexes where queried.
- Every foreign key column gets an index — PostgreSQL does not create them automatically, and missing FK indexes cause table scans on joins and parent deletes.
- **Zero-downtime discipline (expand/contract)** on hot tables:
  - Add columns as nullable (or with a constant default — cheap on PG 11+); backfill in batches; add `NOT NULL`/constraints in a later migration.
  - Never rename a column/table in one step while old code is running: add new → dual-write → migrate readers → drop old.
  - `CREATE INDEX CONCURRENTLY` for indexes on large tables. It can't run inside a transaction, so give that migration a sidecar config file `V9__add_big_index.sql.conf` containing `executeInTransaction=false`.
- Reference data that is genuinely part of the schema (enum lookup tables) may be seeded in migrations with idempotent `INSERT ... ON CONFLICT DO NOTHING`. Environment-specific or test data never goes in migrations.

## Execution

- Applications run Flyway **on startup** (Boot auto-config) — simple and correct for single-writer deployments. If multiple instances deploy simultaneously, Flyway's lock handles it; for large/slow migrations, run Flyway as a separate CI/CD step before rollout instead.
- Local Docker and Testcontainers run the exact same migrations — the migration path is tested on every integration test run. There is no separate dev schema.
- Adopting Flyway on an existing database: `spring.flyway.baseline-on-migrate=true` with an explicit `baseline-version`, once, deliberately.
