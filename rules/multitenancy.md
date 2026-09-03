# Multitenancy Rules (Shared Table + Tenant Discriminator)

Default strategy for multi-tenant projects: **one schema, shared tables, a `tenant_id` discriminator column** (Vlad Mihalcea's "table-based" tier). It scales to many small tenants with zero per-tenant operational cost. Its price is that isolation is enforced by the application, not the database — so every rule here exists to make that enforcement mechanical, not vigilance-based. Schema-per-tenant is the escalation path when a tenant contractually requires hard isolation.

## Mapping

- Every tenant-owned entity carries the tenant discriminator via Hibernate's `@TenantId` (Hibernate 6.3+):

```java
@TenantId
@Column(name = "tenant_id", nullable = false, updatable = false)
private String tenantId;
```

  Hibernate sets it on insert and applies the tenant restriction automatically — never set it by hand, never accept it from a request DTO.
- One `CurrentTenantIdentifierResolver` bean resolves the tenant from request context (JWT claim, header, subdomain) into a request-scoped holder. Tenant resolution happens **once, at the web layer** — services and repositories never take a `tenantId` parameter.
- Native SQL queries bypass `@TenantId` filtering — every native query on a tenant-owned table must include `tenant_id = :tenantId` explicitly, and gets a test proving it.

## Schema (Flyway)

- `tenant_id` is `NOT NULL` on every tenant-owned table.
- Composite indexes on tenant-owned tables **lead with `tenant_id`** (`(tenant_id, created_at)`, not `(created_at)`) — every query is tenant-scoped, so an index that doesn't start with the tenant scans other tenants' entries.
- Unique constraints are per-tenant: `UNIQUE (tenant_id, email)`, never `UNIQUE (email)`.

## Defense in depth (optional, evaluate per project)

- PostgreSQL **Row-Level Security** as a second enforcement layer: `ENABLE ROW LEVEL SECURITY` + a policy on `current_setting('app.tenant_id')`, set per transaction. This moves isolation back into the database — a forgotten predicate returns zero rows instead of another tenant's data. Costs a `SET LOCAL` round-trip per transaction; decide deliberately, like Hypersistence Optimizer in `static-analysis.md`.

## Testing (the checkable part)

- Every multi-tenant project has a **cross-tenant isolation test**: persist data as tenant A, query as tenant B, assert empty — through the real repository path, against Testcontainers PostgreSQL. Same philosophy as the modularity test in `architecture.md`: the boundary fails the build, not the code review.
- Never use tenant IDs as metric tags (`observability.md` — unbounded cardinality).

## References

- Vlad Mihalcea — A beginner's guide to database multitenancy: https://vladmihalcea.com/database-multitenancy/
- Vlad Mihalcea — Hibernate database schema multitenancy (the escalation path): https://vladmihalcea.com/hibernate-database-schema-multitenancy/
- Hibernate ORM — `@TenantId` / discriminator-based multitenancy: https://docs.hibernate.org/orm/current/userguide/html_single/Hibernate_User_Guide.html#multitenacy-discriminator

Note: the `@TenantId` mechanics above come from the Hibernate documentation, not Mihalcea — his articles predate Hibernate 6's discriminator support. Only the strategy tiers, their trade-offs, and the tenant-resolver/context pattern trace to his writing. The Row-Level Security section is an extrapolation of his "play to the database's strengths" principle (`postgresql-aws.md`), not his guidance.
