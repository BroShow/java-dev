# Spring Boot Rules

## Project setup

- Spring Boot **3.x** (latest GA), starters only — don't hand-pick framework versions that the Boot BOM already manages.
- Standard layering: `controller` → `service` → `repository`. Package by feature (`com.acme.order`, `com.acme.customer`), not by layer.
- Every new project sets, from day one:

```yaml
spring:
  jpa:
    open-in-view: false
```

  Open Session in View is an anti-pattern (Vlad Mihalcea): it holds a database connection through view rendering and hides lazy-loading problems until production. Fix fetching in the service layer instead.

## Dependency injection

- **Constructor injection only**, via Lombok:

```java
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepository;
}
```

- Never `@Autowired` on fields. Never setter injection. `final` fields make dependencies explicit and testable.

## Web layer

- Controllers are thin: map HTTP ↔ DTOs, delegate to services. No business logic, no repository access, no `@Transactional` in controllers.
- **Never expose JPA entities from controllers.** Request and response types are records validated with Jakarta Bean Validation (`@Valid`, `@NotNull`, `@Size`, …).
- Centralize error handling in one `@RestControllerAdvice` returning RFC 9457 `ProblemDetail` responses. No try/catch-per-endpoint.
- Every collection endpoint accepts `Pageable` (or explicit limit/offset) and returns a page — never an unbounded list.

## Service layer

- `@Transactional` lives on service methods — one service method = one unit of work.
- Use `@Transactional(readOnly = true)` for all read paths: it lets Hibernate skip dirty checking and snapshot allocation, and lets the pool route to replicas.
- Keep transactions short. Never perform HTTP calls, messaging, or file I/O inside a transaction.

## Configuration & application.yml

- **YAML, not `.properties`** — every project uses `application.yml`.
- Layout: one `application.yml` with sane local-friendly defaults, plus `application-<profile>.yml` per environment (`prod`, `staging`) containing **only the values that differ**. Never duplicate common settings across profile files.
- Profiles for environment differences, not `if` statements in code. Activate via `SPRING_PROFILES_ACTIVE`, never hardcoded.
- **No secrets in any YAML file, ever** — not even "temporarily". Secrets arrive via environment variables or `spring.config.import` from AWS Secrets Manager/Parameter Store (`spring-cloud-aws`). Placeholders reference them: `password: ${DB_PASSWORD}`.
- Type-safe configuration via `@ConfigurationProperties` records with `@Validated` — no scattered `@Value("${...}")`. Fail at startup on bad config, not at first use.
- Keep keys the framework defines in their canonical kebab-case form; group custom properties under a single app prefix (`acme.orders.*`).
- The baseline every new service starts from (merge with the persistence settings in `jpa-hibernate.md` and management settings in `observability.md`):

```yaml
spring:
  application:
    name: <service-name>       # required — feeds metric tags and logs
  jpa:
    open-in-view: false
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true
server:
  shutdown: graceful
```

## Testing

- Prefer slice tests: `@WebMvcTest` for controllers, `@DataJpaTest` for repositories. Reserve `@SpringBootTest` for a few end-to-end flows.
- Use **Testcontainers** with the real database (e.g. PostgreSQL) for persistence tests — H2 hides real SQL behavior and dialect issues.
- Plain JUnit + Mockito for service unit tests; don't boot the context to test business logic.
