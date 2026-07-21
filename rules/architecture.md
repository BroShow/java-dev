# Architecture Rules (Modular Monolith — Oliver Drotbohm / Spring Modulith)

Strong boundaries inside one deployable beat premature distribution. These rules build on the package-by-feature layout in `spring-boot.md`: features become *modules* with enforced boundaries, so the system stays simple now and stays extractable later.

## Module boundaries

- Each top-level feature package is an application module: `com.acme.order`, `com.acme.customer`. The types in the module's root package are its **public API**; implementation details live in an `internal` sub-package (the Spring Modulith convention).
- Modules interact only through each other's API — typically the service (or a dedicated facade) and its DTOs. Never inject another module's repository; never reference another module's entities or `internal` types.
- Keep controllers and repositories package-private where the framework allows — only the module API needs `public`.
- Module dependencies form a DAG — no cycles. A cycle means the modules are cut wrong: merge them or re-cut the boundary.
- Boundaries are enforced by a failing test, not review vigilance. Every project with more than one feature package includes:

```java
@Test
void verifiesModularStructure() {
    ApplicationModules.of(Application.class).verify();
}
```

  Spring Modulith's `verify()` fails the build on cycles and `internal` access; equivalent ArchUnit rules are acceptable where Modulith isn't a dependency.
- Module documentation is generated (Spring Modulith `Documenter`), not hand-drawn — generated diagrams can't drift from the code.

## Modulith first

- Every system starts as a **modular monolith**: one deployable with the boundaries above. Split a module into its own service only for a demonstrated reason — independent scaling, independent team/deploy cadence, a hard isolation requirement — never by default.
- No premature distribution: modules in the same process call each other with plain method calls. No HTTP clients, no queues between modules of one application.
- No speculative flexibility: no interface with a single implementation "so we can swap it later" (see `java-core.md`), no ports-and-adapters ceremony for a future that may not come. A module that only talks through its API *is* the extraction insurance.
- These boundary rules are what makes the monolith modular. Skipping them is what produces the big ball of mud that later forces a rushed microservice rewrite.

## Aggregate design (DDD-lite)

- An aggregate is a root entity plus the children it fully owns (Order → OrderItem). `cascade = CascadeType.ALL` + `orphanRemoval = true` apply only inside an aggregate — same rule as `jpa-hibernate.md`.
- **Within a module**, JPA associations follow `jpa-hibernate.md` unchanged: the lazy `@ManyToOne` child side is still the best mapping.
- **Across module boundaries, reference by ID — never by JPA association.** `Order` stores a `customerId`, not `@ManyToOne Customer`. A cross-module association couples the modules' schemas, invites cross-module lazy-loading, and makes extraction impossible. When order code needs customer data, it asks the customer module's API.
- Repositories exist only for aggregate roots. Children load and change through their root — no `OrderItemRepository`.
- One transaction modifies one aggregate (guideline, not dogma). Two aggregates that must always change atomically are a hint they belong to one module — or that the second change can be eventual.

## References

- Spring Modulith reference documentation: https://docs.spring.io/spring-modulith/reference/
- Spring Modulith project page: https://spring.io/projects/spring-modulith
- jMolecules (architectural concepts as code, Drotbohm et al.): https://github.com/xmolecules/jmolecules
