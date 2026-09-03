# Java Development Rules

These rules apply to ALL Java coding tasks in this workspace. Every Java project here uses **Java 21+, Spring Boot 3.x, and Lombok**, with **PostgreSQL on AWS RDS** as the default database. The persistence rules follow Vlad Mihalcea's High-Performance Java Persistence guidance, the core Java rules follow Joshua Bloch's Effective Java, and the architecture rules follow Oliver Drotbohm's modular-monolith approach (Spring Modulith) — simple by default, efficient by design.

Read and follow all ten rule files before writing Java code:

@rules/java-core.md
@rules/spring-boot.md
@rules/architecture.md
@rules/lombok.md
@rules/jpa-hibernate.md
@rules/postgresql-aws.md
@rules/multitenancy.md
@rules/flyway.md
@rules/observability.md
@rules/static-analysis.md

## Non-negotiables (quick reference)

1. Constructor injection only (`@RequiredArgsConstructor`) — never field injection.
2. `spring.jpa.open-in-view=false` in every new project. No exceptions.
3. All JPA associations are `FetchType.LAZY`, including `@ManyToOne` and `@OneToOne`.
4. ID generation uses `SEQUENCE` — never `IDENTITY`, never `TABLE`.
5. Never put `@Data`, `@EqualsAndHashCode`, or bare `@ToString` on a JPA entity.
6. Read-only screens use DTO projections, not entities.
7. Every list endpoint is paginated.
8. Schema changes go through Flyway migrations — never `ddl-auto=update`.
9. PostgreSQL is the default database: JDBC URL carries `reWriteBatchedInserts=true` and `sslmode=require`; integration tests use Testcontainers with the production PostgreSQL major version, never H2.
10. Database credentials come from AWS Secrets Manager or IAM auth — never committed configuration.
11. Every service ships Actuator + Prometheus metrics from the first commit; actuator endpoints are never internet-facing.
12. Configuration is YAML (`application.yml` + minimal profile overrides); no secrets in any YAML file.
13. Cross-module access goes through a module's public API — never another module's repository, entities, or `internal` package; cross-module references are by ID, and boundaries are verified by a Spring Modulith/ArchUnit test.
14. Formatting is Palantir Java Format via Spotless (`spotless:check` in the build); the Sonar way quality gate blocks on new-code bugs, vulnerabilities, and coverage — code smells stay advisory.
