---
name: java-spring
description: "Use when working on projects involving Java and Spring Boot code."
---

# Java + Spring Boot

## References

- [Spring Boot Reference](https://docs.spring.io/spring-boot/reference/)
- [Spring Framework Reference](https://docs.spring.io/spring-framework/reference/)
- [Spring WebFlux](https://docs.spring.io/spring-framework/reference/web/webflux.html)
- [Java Language Specification](https://docs.oracle.com/javase/specs/)
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [OWASP Dependency-Check](https://jeremylong.github.io/DependencyCheck/)
- [Testcontainers](https://java.testcontainers.org/)
- [Gradle Wrapper](https://docs.gradle.org/current/userguide/gradle_wrapper.html)
- [SpringDoc OpenAPI](https://springdoc.org/)

---

## Code Quality Checks

ALWAYS use `./gradlew`, never bare `gradle`. Check project for existing config first. New projects: Gradle (Groovy DSL), latest LTS Java, latest stable Spring Boot.

```bash
./gradlew spotlessApply                          # 1. Format (Spotless + Google Java Format)
./gradlew compileJava                            # 2. Compile (Error Prone runs here)
./gradlew checkstyleMain spotbugsMain pmdMain    # 3. Static analysis
./gradlew test                                   # 4. Tests
./gradlew dependencyCheckAnalyze                 # 5. Vulnerability scan (OWASP)
```

---

## Project Structure

Hexagonal / ports-and-adapters architecture:

```
project-root/
├── build.gradle
├── gradlew / gradlew.bat
├── gradle/wrapper/
├── src/
│   ├── main/java/com/example/app/
│   │   ├── Application.java
│   │   ├── domain/
│   │   │   ├── model/          # Entities, value objects, enums
│   │   │   ├── port/in/        # Inbound ports (use cases)
│   │   │   ├── port/out/       # Outbound ports (driven)
│   │   │   └── service/        # Domain service implementations
│   │   ├── adapter/
│   │   │   ├── in/web/         # Controllers, request/response DTOs
│   │   │   └── out/persistence/# Repositories (R2DBC for WebFlux, JPA for MVC)
│   │   └── config/             # @Configuration classes
│   ├── main/resources/
│   │   ├── application.yml
│   │   └── application-local.yml
│   └── test/java/com/example/app/
└── README.md
```

---

## Testing

Check project first. New projects: JUnit 5, Mockito, AssertJ, Testcontainers.

- Unit tests for domain services — mock outbound ports
- Integration tests for adapters — use Testcontainers (`@Testcontainers`, `@DynamicPropertySource`)
- Slice tests: `@WebFluxTest` for controllers, `@DataR2dbcTest` for reactive repos (or `@DataJpaTest` for MVC)
- Test naming: `shouldDoX_whenConditionY()`
- Reactive assertions: `StepVerifier`

See `reference/examples.md` for Testcontainers and StepVerifier patterns.

---

## Naming Conventions

| Item | Convention | Example |
|---|---|---|
| Packages | lowercase, dot-separated | `com.example.app.domain` |
| Classes | PascalCase | `UserService`, `OrderController` |
| Interfaces | PascalCase (no I prefix) | `UserRepository`, `PaymentGateway` |
| Methods | camelCase | `findById()`, `processOrder()` |
| Constants | SCREAMING_SNAKE | `MAX_RETRIES`, `DEFAULT_TIMEOUT` |
| Variables | camelCase | `itemCount`, `userName` |
| Enum values | SCREAMING_SNAKE | `ACTIVE`, `PENDING` |
| Test classes | ClassNameTest | `UserServiceTest` |
| Test methods | descriptive with underscores | `shouldReturnUser_whenIdExists()` |

---

## Idioms

- **Constructor injection exclusively.** Never `@Autowired` on fields. `final` fields, single constructor.
- **`@ConfigurationProperties`** for type-safe config. Prefer over `@Value`.
- **`@ControllerAdvice`** for global exception handling — one per bounded context.
- **Records for DTOs and value objects.** Replace mutable POJOs with `record` types.
- **`Optional` only as return type.** Never as a field, parameter, or collection element.
- **Composition over inheritance.** Inject collaborators, don't extend base classes.
- **`@Qualifier` and custom annotations** over string-based bean names.
- **Profile-based configuration.** `application-{profile}.yml` for environment-specific config.
- **Immutable objects wherever possible.** Final fields, no setters, Records.
- **SLF4J + Logback for logging.** Use `LoggerFactory.getLogger()`, never `System.out`. Structured logging for production.
- **Spring Cloud Config Server required for new projects.** All config externalized — secrets use environment placeholders (`${DB_PASSWORD}`) injected at runtime by external automation (control scripts, CI/CD). The application must never contain or log secrets.
- **`@RefreshScope` on config-dependent beans.** Beans that consume `@ConfigurationProperties` values and need runtime refresh must be annotated with `@RefreshScope`. Trigger via `POST /actuator/refresh`.

See `reference/examples.md` for constructor injection, `@ConfigurationProperties`, and config server patterns.

---

## Design Patterns

- **Hexagonal architecture** — Domain knows nothing about adapters. Ports define boundaries.
- **Strategy** — Inject implementations via `@Qualifier`.
- **Template method** — Rare; only for framework patterns requiring inheritance. Prefer composition.
- **Observer** — `ApplicationEventPublisher` for decoupled domain events.
- **Factory** — `@Bean` methods in `@Configuration` classes.
- **Decorator** — Spring AOP / proxies for cross-cutting concerns.
- **Repository** — Spring Data interfaces for data access.
- **CQRS** — Separate read/write models when query and command needs diverge.

---

## Anti-Patterns

**NEVER do these:**

- **Project Lombok — FORBIDDEN.** Never use `@Data`, `@Value`, `@Builder`, `@Getter`, `@Setter`, or any Lombok annotation. Use Records, manual constructors, IDE generation. Migrate existing Lombok usage away.
- **Field injection** — `@Autowired` on fields. Use constructor injection.
- **God classes** — Split services by use case.
- **Catching `Exception`/`Throwable` broadly** — Catch specific exceptions only.
- **Business logic in controllers** — Controllers delegate to domain services only.
- **Exposing domain entities as API responses** — Use DTOs (Records).
- **Circular dependencies** — Redesign with events/mediator or `ObjectProvider<T>`.
- **N+1 queries** — Use `@EntityGraph`, join fetch, or batch fetching.
- **Hardcoded configuration** — Externalize via Spring Cloud Config Server. Secrets use `${PLACEHOLDER}` injected by automation, never committed to source.
- **Secrets in source or logs** — Never hardcode, log, or commit secrets.
- **Mutable DTOs** — Use Records. No setters.

---

## Error Handling

- `@ControllerAdvice` + `@ExceptionHandler` for centralized handling
- Domain-specific exception hierarchy rooted in `DomainException`
- RFC 7807 Problem Details (`ProblemDetail` in Spring 6+) for HTTP errors
- Never swallow exceptions — log and rethrow or transform
- Log levels: `ERROR` unrecoverable, `WARN` recoverable, `DEBUG` diagnostics
- Reactive: `Mono.error()` / `Flux.error()` for propagation

See `reference/examples.md` for `@RestControllerAdvice` and domain exception patterns.

---

## Reactive Programming (WebFlux)

Check project for existing stack. New projects use Spring WebFlux.

- **Never block** inside reactive chains. No `.block()`, no blocking I/O.
- **`.flatMap()`** for async composition. `.map()` for synchronous transforms only.
- **Backpressure** — `.onBackpressureBuffer()`, `.onBackpressureDrop()` as needed.
- **`Schedulers.boundedElastic()`** — Wrap unavoidable blocking calls with `.subscribeOn()`.
- **`WebClient`** for HTTP calls. Never `RestTemplate` in reactive code.
- **`StepVerifier`** for testing reactive streams.
- Avoid blocking persistence in reactive flows — use R2DBC, not JPA.

See `reference/examples.md` for WebFlux controller, reactive composition, and blocking call wrapper patterns.

---

## Documentation

- All public classes and methods must have Javadoc
- Javadoc on interfaces, not implementations (unless adding behavior)
- `@param`, `@return`, `@throws` for non-trivial methods. Package docs in `package-info.java`.

### SpringDoc OpenAPI + Swagger UI (Required)

New projects must include `springdoc-openapi`. Swagger UI serves at `/swagger-ui.html`.

```groovy
implementation 'org.springdoc:springdoc-openapi-starter-webflux-ui:2.+'   // WebFlux
implementation 'org.springdoc:springdoc-openapi-starter-webmvc-ui:2.+'    // MVC
```

- Annotate app with `@OpenAPIDefinition` (`@Info` for title, version, description)
- Annotate controllers with `@Tag` to group by domain
- Annotate endpoints with `@Operation`, `@ApiResponse`, `@Parameter`
- Annotate DTOs with `@Schema` (description, examples)
- Every exposed endpoint must have OpenAPI annotations

See `reference/examples.md` for SpringDoc annotation patterns and `application.yml` config.

---

## Dependencies

- Check `build.gradle` before adding any dependency
- Use Spring Boot BOM — do not specify versions for managed dependencies
- Pin versions for non-managed dependencies
- Run `./gradlew dependencyCheckAnalyze` after adding new dependencies
- Prefer Spring ecosystem libraries (Spring Data, Spring Security, Spring Cloud)

---

## Performance Considerations

- Connection pooling: HikariCP (Spring Boot default)
- Thread pools: configure for WebFlux (`reactor.netty` defaults)
- Caching: `@Cacheable` for frequently accessed, rarely changing data
- Pagination for collection endpoints (`Pageable` / reactive equivalents)
- Projections/DTOs in queries — never fetch full entities for partial data
- Database indexing aligned with query patterns

---

## Troubleshooting

For legacy migration guidance, see `reference/legacy-migration.md`.

### Remote Debugging

```bash
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005 -jar app.jar
./gradlew bootRun -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=y,address=*:5005"
```

### Thread & Heap Dumps

```bash
jps -l                                    # Find PID
jcmd <pid> Thread.print > threads.txt     # Thread dump (deadlocks)
jcmd <pid> GC.heap_dump heap.hprof        # Heap dump (memory leaks)
jcmd <pid> GC.class_histogram | head -20  # Quick memory by class
```

### Actuator Diagnostics

Enable: `management.endpoints.web.exposure.include=env,health,info,beans,conditions,mappings,refresh`

**Security — required:**
- All actuator endpoints except `/actuator/health` and `/actuator/info` MUST be secured with Spring Security.
- Use dedicated actuator credentials (in-memory user), separate from application auth. Credentials externalized via `${ACTUATOR_USERNAME}` / `${ACTUATOR_PASSWORD}`.
- Sanitize sensitive values in `/actuator/env`: set `management.endpoint.env.show-values=WHEN_AUTHORIZED`.
- Never expose actuator endpoints publicly in production without authentication.
- Use `management.server.port` to bind actuator to an internal-only port when possible.

| Endpoint | Use |
|---|---|
| `/actuator/env` | Property sources and precedence |
| `/actuator/conditions` | Why auto-config beans were/weren't created |
| `/actuator/beans` | All registered beans |
| `/actuator/health` | DB connectivity, disk, custom checks |
| `/actuator/info` | App version, git info, custom metadata |
| `/actuator/mappings` | All request mappings |
| `/actuator/refresh` | Reload `@RefreshScope` beans from Config Server |

See `reference/examples.md` for actuator security configuration.

### Common Startup Failures

| Error | Root Cause | Fix |
|---|---|---|
| `BeanCreationException` | Constructor/init failed | Check `Caused by:` — NPE, missing config, DB down |
| `NoSuchBeanDefinitionException` | Missing annotation or scan | Verify `@Component`/`@Service`, `@ComponentScan` |
| `UnsatisfiedDependencyException` | Can't wire bean | Check type, `@Qualifier`, conditionals |
| `BeanCurrentlyInCreationException` | Circular dependency | Refactor or use `ObjectProvider<T>` |
| `NoUniqueBeanDefinitionException` | Multiple beans of same type | `@Primary` or `@Qualifier` |
| `ConfigurationPropertiesBindException` | Type mismatch | Check `Origin:` in error for file and line |
| `PortInUseException` | Port bound | `lsof -i :<port>` |

### Legacy Version Detection

```bash
./gradlew dependencies | grep spring-boot     # Spring Boot version
./gradlew dependencies | grep javax           # javax = pre-3.x
./gradlew dependencies | grep jakarta         # jakarta = 3.x+
```

If `javax.*` imports with Spring Boot 3.x → incomplete migration. See `reference/legacy-migration.md`.

---

## Design Principles

- **SOLID** — Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, Dependency Inversion
- **DRY** — Don't repeat yourself, but don't extract prematurely.
- **KISS** — Straightforward solutions over clever abstractions.
- **YAGNI** — Don't build for hypothetical requirements.
- **Composition over inheritance** — Inject via constructor.
- **Program to interfaces** — Depend on ports, not implementations.
- **Separation of concerns** — Domain in domain layer, infrastructure in adapters.
- **Fail fast** — Validate early. Domain exceptions at the boundary.

---

## Checklist Before Completion

- [ ] Code compiles: `./gradlew compileJava`
- [ ] Code is formatted: `./gradlew spotlessApply`
- [ ] Static analysis passes: `./gradlew checkstyleMain spotbugsMain pmdMain`
- [ ] All tests pass: `./gradlew test`
- [ ] Vulnerability check: `./gradlew dependencyCheckAnalyze`
- [ ] No Lombok usage anywhere
- [ ] Constructor injection used exclusively
- [ ] All public classes and methods have Javadoc
- [ ] Domain logic lives in `domain/service/`, not adapters
- [ ] DTOs use Records, not mutable classes
- [ ] Exceptions are domain-specific, not generic
- [ ] Reactive chains do not block
- [ ] No business logic in controllers
- [ ] Configuration externalized via Spring Cloud Config Server (new projects)
- [ ] Secrets use environment placeholders — no hardcoded values in source
- [ ] No secrets logged or exposed in error responses
- [ ] All endpoints have SpringDoc OpenAPI annotations (`@Operation`, `@Tag`, `@ApiResponse`, `@Schema` on DTOs)
