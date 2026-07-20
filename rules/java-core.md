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

## General style

- Small, single-purpose methods. If a method needs a comment to explain *what* it does, split or rename it.
- Prefer composition over inheritance; avoid deep class hierarchies.
- Streams for transformation pipelines; plain loops when logic has side effects or early exits — whichever reads simpler.
- No premature abstraction: no interfaces with a single implementation unless a framework requires it.
