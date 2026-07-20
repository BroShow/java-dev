# Java Development Rules

These rules apply to ALL Java coding tasks in this workspace. Every Java project here uses **Java 21+, Spring Boot 3.x, and Lombok**, with **PostgreSQL on AWS RDS** as the default database. The persistence rules follow Vlad Mihalcea's High-Performance Java Persistence guidance — simple by default, efficient by design.

Read and follow all five rule files before writing Java code:

@rules/java-core.md
@rules/spring-boot.md
@rules/lombok.md
@rules/jpa-hibernate.md
@rules/postgresql-aws.md

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
