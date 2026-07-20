# JPA / Hibernate Rules (High-Performance Java Persistence — Vlad Mihalcea)

Fetching too much data is the #1 JPA performance problem. Every rule here exists to fetch exactly what a use case needs — no more — and to let the database do what it's good at.

## Fetching

- **Every association is `FetchType.LAZY`.** `@ManyToOne` and `@OneToOne` are EAGER by default in JPA — always override them:

```java
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "customer_id")
private Customer customer;
```

  EAGER is a per-mapping decision you can never undo per-query; LAZY can always be upgraded per-query.
- **Kill N+1 at the query level**, not with EAGER: use `JOIN FETCH` in JPQL or `@EntityGraph` on the repository method when the use case needs the association.
- Never `JOIN FETCH` **two collections** in one query (Cartesian product) and never combine a collection `JOIN FETCH` with pagination (Hibernate paginates in memory — `HHH90003004` warning).
- **DTO projections for every read-only use case.** If the data is only displayed and never modified, select a record directly:

```java
@Query("""
    select new com.acme.order.OrderSummary(o.id, o.total, c.name)
    from Order o join o.customer c
    where o.status = :status
    """)
List<OrderSummary> findSummaries(@Param("status") OrderStatus status);
```

  Entities are for write flows, where dirty checking earns its cost. DTOs fetch fewer columns, skip the persistence context, and can't throw `LazyInitializationException`.
- **Always paginate.** Fetching more rows than the UI can display is pure waste. Any query that can return many rows takes a `Pageable`.

## Identifiers

- Use `SEQUENCE` generation — never `IDENTITY` (it disables JDBC batching for inserts because Hibernate must execute the INSERT to get the ID) and never `TABLE` (row-lock contention, worst performer):

```java
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
@SequenceGenerator(name = "order_seq", sequenceName = "order_seq", allocationSize = 50)
private Long id;
```

- `allocationSize = 50` (the pooled optimizer) cuts sequence round-trips 50×.
- On PostgreSQL (our default database), never map columns as `SERIAL`/`BIGSERIAL` — they tie you to `IDENTITY` semantics. Flyway migrations declare `BIGINT` + an explicit sequence that matches the `@SequenceGenerator`.
- Only if a project is forced onto MySQL (no sequences) is `IDENTITY` the pragmatic fallback — accept the lost insert batching.

## equals / hashCode

Must stay consistent across ALL entity state transitions (transient → managed → detached → removed). The standard implementation:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (!(o instanceof Order other)) return false;   // instanceof handles proxies
    return id != null && id.equals(other.getId());   // getters, not fields — proxies again
}

@Override
public int hashCode() {
    return getClass().hashCode();   // constant: id-based hash would change on persist
}
```

If the entity has a truly immutable, unique **natural key** (ISBN, account number), use that instead of the ID. Never use Lombok to generate these (see `lombok.md`).

## Association mapping

- The `@ManyToOne` child side is the best mapping. Map a bidirectional `@OneToMany` (with `mappedBy`) only when the parent genuinely needs the collection, and always with helper methods that keep both sides in sync:

```java
public void addItem(OrderItem item) {
    items.add(item);
    item.setOrder(this);
}
```

- **Never use a unidirectional `@OneToMany`** — it produces a join table or extra UPDATE statements.
- `@ManyToMany`: use `Set`, not `List` (with `List`, removing one element deletes all join rows and re-inserts the rest). Cascade only `PERSIST` and `MERGE` — never `REMOVE`/`ALL`.
- `cascade = CascadeType.ALL` + `orphanRemoval = true` belongs only on parent → child compositions the parent fully owns (Order → OrderItem).
- Add `@Version` for optimistic locking on every entity that can be concurrently updated.
- Use `@DynamicUpdate` only for entities with many columns or heavy/LOB columns; the default full UPDATE caches better as a prepared statement.

## Batching & Hibernate settings

Every project's `application.yml` includes:

```yaml
spring:
  jpa:
    open-in-view: false
    properties:
      hibernate:
        jdbc.batch_size: 30
        order_inserts: true
        order_updates: true
        query.in_clause_parameter_padding: true
```

- `batch_size` + `order_inserts`/`order_updates` turn N inserts into N/30 round-trips.
- Always add `reWriteBatchedInserts=true` to the PostgreSQL JDBC URL — the driver rewrites batches into multi-row INSERTs (see `postgresql-aws.md`).
- `in_clause_parameter_padding` improves execution-plan cache hit rates for `IN` queries.
- On Oracle (if ever used), raise `hibernate.jdbc.fetch_size` — the driver default is only 10.

## Bulk & write operations

- Mass updates/deletes use bulk JPQL (`@Modifying(clearAutomatically = true) @Query("update ...")`) or SQL — never load-modify-save loops over entities.
- `saveAll` with batching for bulk inserts; flush and clear the persistence context periodically for very large batches.
- Prefer `getReferenceById` over `findById` when you only need a reference for a foreign key — it avoids a SELECT entirely.

## Schema & verification

- Schema is owned by **Flyway** migrations. `spring.jpa.hibernate.ddl-auto` is `validate` (or `none`) everywhere — `update`/`create` never touch a shared environment.
- In development, log SQL through the datasource proxy or set `logging.level.org.hibernate.orm.jdbc.bind=trace` when debugging — and **look at the SQL Hibernate generates** for any new query path.
- Persistence tests should assert statement counts where N+1 regressions are likely (e.g. Hypersistence Utils / datasource-proxy `assertSelectCount`).
- Consider the second-level cache only after fetching, batching, and the database's own buffer pool are already tuned — it is a last-resort optimization, not a default.

## References

- Vlad Mihalcea — Hibernate performance tuning tips: https://vladmihalcea.com/hibernate-performance-tuning-tips/
- The Open Session In View anti-pattern: https://vladmihalcea.com/the-open-session-in-view-anti-pattern/
- Spring Data JPA DTO projections: https://vladmihalcea.com/spring-jpa-dto-projection/
- equals/hashCode for entities: https://vladmihalcea.com/the-best-way-to-implement-equals-hashcode-and-tostring-with-jpa-and-hibernate/
