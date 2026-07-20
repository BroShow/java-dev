# Lombok Rules

Lombok removes boilerplate — but a few annotations are dangerous on JPA entities. Follow these rules exactly.

## Allowed everywhere

| Annotation | Use for |
|---|---|
| `@Getter` / `@Setter` | Entities and mutable beans |
| `@RequiredArgsConstructor` | Constructor injection in Spring components |
| `@NoArgsConstructor` / `@AllArgsConstructor` | Entities (JPA needs no-arg), builders |
| `@Builder` | Complex object construction (services, test data) |
| `@Slf4j` | Logging — never declare a `Logger` field manually |
| `@Value` | Immutable non-entity classes (prefer a `record` when possible) |

## Forbidden on JPA entities

- **`@Data` — never on an entity.** It bundles `@EqualsAndHashCode` and `@ToString`, both of which break entities:
  - `@EqualsAndHashCode` uses all fields, so `hashCode` changes when the generated ID is assigned — an entity added to a `HashSet` before persisting can no longer be found afterwards, and equality breaks across entity state transitions (Vlad Mihalcea's consistency rule).
  - `@ToString` (and `@EqualsAndHashCode`) touch every field, which triggers lazy-loading of associations — surprise queries or `LazyInitializationException` from a log statement.
- On entities use `@Getter @Setter @NoArgsConstructor` and write `equals`/`hashCode` by hand (see `jpa-hibernate.md` for the required implementation).
- If an entity needs `toString`, use `@ToString(onlyExplicitlyIncluded = true)` and `@ToString.Include` only on basic (non-association) fields.
- `@Builder` on entities is acceptable, but keep the JPA-required `@NoArgsConstructor` and never let the builder bypass association helper methods.

## DTOs: prefer records

For immutable DTOs, prefer a plain Java `record` over `@Value`/`@Data` classes. Use Lombok where mutability or builders are genuinely needed.

## Project configuration

Every project gets a root `lombok.config`:

```
config.stopBubbling = true
lombok.addLombokGeneratedAnnotation = true
```

The second line makes JaCoCo and other tools skip generated code in coverage reports.
