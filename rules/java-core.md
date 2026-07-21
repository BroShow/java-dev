# Java Core Rules

## Language & toolchain

- Target **Java 21** (LTS) or newer. Use Maven unless the project already uses Gradle.
- Prefer modern language features when they simplify code: records, sealed interfaces, pattern matching for `switch`, text blocks, enhanced `instanceof`.
- Use `var` only for local variables where the type is obvious from the right-hand side.

## Types & immutability

- **Records for all DTOs, API request/response objects, and value objects.** A record with validation annotations beats a Lombok class for anything immutable that isn't a JPA entity.
- Make classes and fields `final` by default; expose mutability only where genuinely needed.
- Collections returned from methods should be unmodifiable (`List.copyOf`, `Collections.unmodifiableList`) or streams.
- Use `Optional` only as a return type — never for fields or method parameters.

## Time & money

- Use `java.time` exclusively (`Instant`, `LocalDate`, `OffsetDateTime`). Never `java.util.Date` or `Calendar`.
- Store timestamps in UTC; convert to user time zones at the presentation layer only.
- Use `BigDecimal` for money (with explicit scale and `RoundingMode`) — never `double`/`float`.

## Errors & nullability

- Throw specific exceptions; create small domain exceptions (e.g. `OrderNotFoundException extends RuntimeException`) rather than throwing generic `RuntimeException`.
- Never swallow exceptions. Never `catch (Exception e) { log; return null; }`.
- Don't return `null` from public methods — return `Optional`, an empty collection, or throw.
- Validate method arguments at public API boundaries (`Objects.requireNonNull`, guard clauses).

## Object contracts (Effective Java)

- Prefer records for value types — they get a correct `equals`/`hashCode`/`toString` for free. Hand-write these methods only when a class genuinely can't be a record. (JPA entities follow the separate implementation in `jpa-hibernate.md`.)
- Override `equals` and `hashCode` **as a pair**, never one alone, honoring the contract (reflexive, symmetric, transitive, consistent) and basing both on the same fields. `Objects.hash(...)` is fine.
- Never override `equals` on a mutable class whose instances are used as `HashMap` keys or `HashSet` elements — mutation after insertion silently loses the entry.
- Value types with a natural ordering implement `Comparable`; build `compareTo` with `Comparator.comparing(...).thenComparing(...)` chains — never subtraction (overflow). Keep ordering consistent with `equals`, and document when it can't be (the `BigDecimal` trap).
- Every value type has a useful `toString` — it ends up in logs and error messages.

## API design (Effective Java)

- Use a static factory method instead of a constructor when a name adds clarity (`Money.of(amount, currency)`) or instances can be shared/cached.
- More than ~4 parameters — especially optional ones or several of the same type — means a builder (`@Builder` per `lombok.md`), not a telescoping constructor.
- Minimize accessibility: classes package-private unless they are part of a module's public API (see `architecture.md`); members as private as possible.
- Design for inheritance or prohibit it: classes stay `final` by default (see above); a deliberately extendable class must document how its methods call each other.
- Declare parameter and return types as interfaces (`List`, `Map`) — never implementation types (`ArrayList`).
- Avoid overload sets whose resolution hinges on autoboxing or generics — give the methods distinct names instead.

## Generics & collections

- No raw types, ever. When a cast is provably safe, `@SuppressWarnings("unchecked")` on the smallest possible scope with a comment stating why it's safe.
- Public APIs that consume or produce collections use bounded wildcards per PECS — `Collection<? extends T>` from producers, `Collection<? super T>` for consumers. Return types stay exact; no wildcards in returns.
- Prefer `List<T>` over `T[]` in APIs — arrays are covariant and fail at runtime; generics fail at compile time.
- Use `EnumSet`/`EnumMap` for enum-driven logic — never ordinal-indexed arrays or bit fields.
- Generic varargs only with `@SafeVarargs` and only when actually safe; otherwise take a `List<T>`.

## Lambdas & streams

- Streams for transformation pipelines; plain loops when logic has side effects, early exits, or checked exceptions — whichever reads simpler. Never force a nontrivial algorithm into one giant stream chain (Bloch's own caution).
- Prefer method references when they're clearer than the lambda (`Order::total`), the lambda when it's clearer than the reference.
- Lambdas stay short — a few lines at most. Anything longer becomes a named private method, used via method reference.
- Use the standard `java.util.function` interfaces; don't define custom functional interfaces that duplicate them.

## General style

- Small, single-purpose methods. If a method needs a comment to explain *what* it does, split or rename it.
- Prefer composition over inheritance; avoid deep class hierarchies.
- No premature abstraction: no interfaces with a single implementation unless a framework requires it.

## References

- Joshua Bloch — *Effective Java*, 3rd edition (object contracts: Items 10–14; factories & builders: Items 1–2; accessibility & inheritance: Items 15–19; generics: Items 26–33; lambdas & streams: Items 42–48)
