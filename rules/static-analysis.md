# Static Analysis, Formatting & Coverage Rules (Spotless, SonarQube, JaCoCo)

Machines own style and mechanical quality; the rule files stay reserved for what tools can't check. No formatting or lint debates in review — the build settles them.

## Formatting — Palantir Java Format via Spotless

- Every project formats with **palantir-java-format** (120-column, 4-space indent, zero configuration knobs) applied through the Spotless Maven plugin. The formatter's output *is* the style — nobody argues with it, and no rule file discusses formatting.
- `spotless:check` is bound to the build so CI fails on unformatted code; developers run `mvn spotless:apply` before committing (or use the palantir-java-format IntelliJ plugin):

```xml
<plugin>
    <groupId>com.diffplug.spotless</groupId>
    <artifactId>spotless-maven-plugin</artifactId>
    <version>${spotless.version}</version>
    <configuration>
        <java>
            <palantirJavaFormat/>
        </java>
    </configuration>
    <executions>
        <execution>
            <goals>
                <goal>check</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- No per-project style overrides. If Palantir format produces something ugly, restructure the code, not the formatter.

## SonarQube — Sonar way defaults, blocking core

- Quality profile: the built-in **Sonar way** defaults. No custom profile up front — curating rules before feeling their noise is its own overengineering.
- Quality gate follows Clean as You Code and applies to **new code**: zero new Bugs, zero new Vulnerabilities, all new Security Hotspots reviewed, and new-code coverage ≥ 80% (see JaCoCo below). **Code Smells are advisory** — tracked and visible, never build-breaking.
- Method and class size are governed by **cognitive complexity** (rule S3776, default threshold 15) — not line counts. It measures nesting and branching, which is what actually hurts readability, and it resolves the "how small is too small" debate mechanically.
- Deactivating a rule or changing a threshold requires a written reason, recorded where the change lives (quality profile description or a comment in `sonar-project.properties`). No silent disables.
- In-code suppressions use `@SuppressWarnings("java:SXXXX")` at the narrowest possible scope with a comment justifying it.
- Legacy code is not the gate's problem: the gate watches new code, so adopting Sonar on an old codebase never means boiling the ocean first.
- Developers run SonarQube for IDE (SonarLint) in connected mode so findings appear at edit time, not at PR time.

## JaCoCo — test coverage

- Every project runs the `jacoco-maven-plugin` with every test build; the local HTML report lands at `target/site/jacoco/index.html`:

```xml
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>${jacoco.version}</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
            </goals>
        </execution>
        <execution>
            <id>report</id>
            <phase>test</phase>
            <goals>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

- **No hard coverage threshold in Maven.** Enforcement is Sonar's new-code coverage condition (80%) — legacy modules never block a build, and the number that matters is coverage of the code you just wrote.
- `lombok.config`'s `addLombokGeneratedAnnotation` (see `lombok.md`) already keeps Lombok-generated code out of the numbers. Beyond that, exclude only genuinely generated sources — never exclude code to hit a number.
- Coverage is a signal, not a goal: an uncovered branch is a question to answer, and an assertion-free test written to satisfy the gate is a violation of these rules, not compliance with them.

## Persistence static analysis — Hypersistence Optimizer (optional)

Spotless and Sonar don't know anything about JPA mappings: an EAGER `@ManyToOne`, an `IDENTITY` id, or a missing `removeItem` passes every check in this file. **Hypersistence Optimizer** (Vlad Mihalcea, commercial) closes that gap — it inspects the `EntityManagerFactory` at startup and reports exactly the rules in `jpa-hibernate.md` as CRITICAL/MAJOR events.

- **Optional, evaluate per project** — it is paid, so it is a recommendation, not a non-negotiable. Worth it on a project large enough that mapping review by eye has stopped being reliable.
- If adopted, it is one bean, and it belongs in a **test** so the failure lands at development time rather than in a production log:

```java
@Bean
public HypersistenceOptimizer hypersistenceOptimizer(EntityManagerFactory entityManagerFactory) {
    return new HypersistenceOptimizer(new JpaConfig(entityManagerFactory));
}
```

- Treat it the way the modularity test in `architecture.md` is treated: assert on zero CRITICAL events so the build fails, rather than logging events nobody reads. Any event deliberately accepted gets a recorded reason, same as a deactivated Sonar rule.
- Nothing here is a substitute for the rules themselves — the tool checks mappings, not whether a read path uses a DTO projection or is paginated.

## References

- SonarSource — Cognitive Complexity white paper: https://www.sonarsource.com/resources/cognitive-complexity/
- Palantir Java Format: https://github.com/palantir/palantir-java-format
- Spotless: https://github.com/diffplug/spotless
- Vlad Mihalcea — Tuning Spring Petclinic with Hypersistence Optimizer: https://vladmihalcea.com/spring-petclinic-hypersistence-optimizer/
