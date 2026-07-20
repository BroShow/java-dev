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
- **Use keyset (seek) pagination for deep paging and infinite scroll.** OFFSET scans and discards every skipped row — page 1,000 is dramatically slower than page 1, and rows shift between requests as data changes. For "next page" navigation over large or live datasets, seek on the last-seen key via Spring Data's `WindowIterator`/`ScrollPosition.keyset()`. Plain offset `Pageable` is fine for small, admin-style result sets with numbered pages.

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
- **Never use random UUIDv4 as a primary key.** Random bytes destroy B+Tree index locality: every insert lands on a random page, bloating the index and evicting hot pages from the buffer pool.
- **When globally unique or externally exposed IDs come up, consider TSID** (not mandatory — evaluate per project). Mihalcea's recommendation is a 64-bit time-sorted identifier stored as `bigint`: half the size of a UUID, monotonically increasing, so it behaves like a sequence in the index. His [hypersistence-tsid](https://github.com/vladmihalcea/hypersistence-tsid) library implements it. If the ID must be a real UUID (external contract), use time-ordered UUIDv7, never v4. If neither constraint exists, plain `SEQUENCE` remains the default.

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
- **`@OneToOne`: map the child side with `@MapsId`** so the child shares the parent's primary key:

```java
@Entity
public class OrderDetails {
    @Id
    private Long id;

    @OneToOne(fetch = FetchType.LAZY)
    @MapsId
    @JoinColumn(name = "order_id")
    private Order order;
}
```

  No separate FK column or index, and the parent can load the child by its own ID. Avoid the parent-side `@OneToOne(mappedBy = ...)` unless truly needed — the parent side cannot be lazy without bytecode enhancement, so it triggers an extra SELECT per parent.
- Use `@DynamicUpdate` only for entities with many columns or heavy/LOB columns; the default full UPDATE caches better as a prepared statement.

## Enum mapping

Pick the mapping deliberately — never leave `@Enumerated` unspecified (the default is ORDINAL, silently):

- Default choice: `@Enumerated(EnumType.STRING)` — self-documenting in queries and safe to reorder enum constants. Pair with a `CHECK` constraint (or PostgreSQL enum type) in the Flyway migration so the database rejects invalid values.
- `EnumType.ORDINAL` (with a `smallint` column) only for high-volume tables where the storage difference is measurable — and then constants may **never** be reordered or removed, only appended. Document this on the enum itself.

## Concurrency & locking

- **Every entity that users can concurrently edit gets a `@Version` field.** Without it, the last write silently wins and the first user's update is lost — optimistic locking is a correctness requirement, not an optimization. This matters even more with read replicas and long user think-time between read and save.
- Handle `OptimisticLockException` deliberately at the service boundary: retry idempotent system operations; surface a conflict (HTTP 409) for user edits so the user re-reads before overwriting.
- Detached DTO → update flows must carry the version through the round-trip (include it in the form/DTO) or optimistic locking silently checks the wrong version.
- Reach for pessimistic locks (`PESSIMISTIC_WRITE`, PostgreSQL `SELECT ... FOR UPDATE`) only when contention is proven and retries are unacceptable; for cross-instance coordination use advisory locks (see `postgresql-aws.md`).

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
- The best UUID type for a primary key (TSID): https://vladmihalcea.com/uuid-database-primary-key/
- Keyset pagination: https://vladmihalcea.com/keyset-pagination-jpa-hibernate/ and https://vladmihalcea.com/spring-data-windowiterator/
- The best way to map a @OneToOne: https://vladmihalcea.com/best-way-onetoone-optional/
