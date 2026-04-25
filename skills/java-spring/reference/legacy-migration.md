# Java + Spring Boot Legacy Migration Reference

Detailed debugging and migration guidance for legacy Spring Boot and Java versions.

## Version Compatibility Matrix

| Spring Boot | Min Java | Spring Framework | Jakarta EE | Hibernate |
|---|---|---|---|---|
| 2.7.x | 8 | 5.3.x | javax (EE 8) | 5.6.x |
| 3.0.x | 17 | 6.0.x | jakarta (EE 9) | 6.1.x |
| 3.1.x | 17 | 6.0.x | jakarta (EE 9) | 6.2.x |
| 3.2.x | 17 | 6.1.x | jakarta (EE 10) | 6.4.x |

Upgrade path: Java 8 → 11 → 17 → 21, Spring Boot 2.7 → 3.0 → 3.2+

---

## javax → jakarta Namespace Migration

Spring Boot 3.0+ requires Jakarta EE 9+. All `javax.*` packages move to `jakarta.*`.

### Package Mapping

| javax.* | jakarta.* |
|---|---|
| `javax.servlet.*` | `jakarta.servlet.*` |
| `javax.servlet.http.*` | `jakarta.servlet.http.*` |
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.annotation.PostConstruct` | `jakarta.annotation.PostConstruct` |
| `javax.annotation.PreDestroy` | `jakarta.annotation.PreDestroy` |
| `javax.transaction.*` | `jakarta.transaction.*` |
| `javax.jms.*` | `jakarta.jms.*` |
| `javax.mail.*` | `jakarta.mail.*` |
| `javax.websocket.*` | `jakarta.websocket.*` |
| `javax.xml.bind.*` | `jakarta.xml.bind.*` |
| `javax.json.*` | `jakarta.json.*` |
| `javax.ws.rs.*` | `jakarta.ws.rs.*` |

### Detection

```bash
# Find javax imports in source
find src -name "*.java" -exec grep -l "import javax\." {} \;

# Find javax in JARs
find . -name "*.jar" -exec sh -c 'jar tf "$1" | grep "^javax/" && echo "=== $1 ==="' _ {} \;

# Verify migration completeness
jar tf application.jar | grep -c "javax\."    # Should be 0 for Spring Boot 3.x
```

### Property Migrator

Add temporarily during migration to detect renamed properties:

```groovy
dependencies {
    runtimeOnly 'org.springframework.boot:spring-boot-properties-migrator'
}
```

Remove after migration is complete — it adds startup overhead.

---

## Removed APIs

### WebSecurityConfigurerAdapter (removed in Spring Security 6.0 / Boot 3.0)

```java
// REMOVED
@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.authorizeRequests()
            .antMatchers("/public/**").permitAll()
            .anyRequest().authenticated();
    }
}

// REPLACEMENT
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.authorizeHttpRequests(auth -> auth
            .requestMatchers("/public/**").permitAll()
            .anyRequest().authenticated()
        );
        return http.build();
    }
}
```

Note: `antMatchers()` → `requestMatchers()` in Spring Security 6.0+.

### WebMvcConfigurerAdapter (removed in Spring 5.0)

Already an interface in Spring 5.x — just implement `WebMvcConfigurer` directly.

### EmbeddedServletContainerFactory (removed in Boot 2.0)

```java
// REMOVED
@Bean
public EmbeddedServletContainerCustomizer customizer() { ... }

// REPLACEMENT
@Bean
public WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> customizer() {
    return factory -> {
        factory.setPort(9000);
        factory.setContextPath("/api");
    };
}
```

### RestTemplate (deprecated in Spring 6.1 / Boot 3.2)

```java
// DEPRECATED — still works but receives no new features
RestTemplate restTemplate = new RestTemplateBuilder().build();

// REPLACEMENT (blocking)
RestClient restClient = RestClient.builder().baseUrl("https://api.example.com").build();
String result = restClient.get().uri("/users/{id}", 1).retrieve().body(String.class);

// REPLACEMENT (reactive — preferred for WebFlux)
WebClient webClient = WebClient.builder().baseUrl("https://api.example.com").build();
Mono<String> result = webClient.get().uri("/users/{id}", 1).retrieve().bodyToMono(String.class);
```

---

## Renamed Properties

| Spring Boot 2.x | Spring Boot 3.x |
|---|---|
| `spring.resources.*` | `spring.web.resources.*` |
| `spring.mvc.locale-resolver` | `spring.web.locale-resolver` |
| `spring.data.mongodb.grid-fs-database` | `spring.data.mongodb.gridfs.database` |
| `spring.jpa.hibernate.use-new-id-generator-mappings` | Removed (always true) |

Hibernate dialect is auto-detected in Spring Boot 3.x — remove `spring.jpa.properties.hibernate.dialect` unless overriding.

---

## Java Version Migration Issues

### Java 8 → 17

#### Module System (JPMS) — Reflection Access

Many libraries rely on deep reflection. Add `--add-opens` flags when you see:

```
java.lang.IllegalAccessException: class X cannot access class Y from module Z
```

Common flags needed:

```bash
--add-opens java.base/java.lang=ALL-UNNAMED
--add-opens java.base/java.util=ALL-UNNAMED
--add-opens java.base/java.util.concurrent=ALL-UNNAMED
--add-opens java.base/java.io=ALL-UNNAMED
--add-opens java.base/java.nio=ALL-UNNAMED
--add-opens java.base/sun.nio.ch=ALL-UNNAMED
--add-opens jdk.unsupported/sun.misc=ALL-UNNAMED
```

In Gradle:

```groovy
tasks.withType(Test) {
    jvmArgs = [
        '--add-opens', 'java.base/java.util=ALL-UNNAMED',
        '--add-opens', 'java.base/java.lang=ALL-UNNAMED'
    ]
}

bootRun {
    jvmArgs = [
        '--add-opens', 'java.base/java.util=ALL-UNNAMED',
        '--add-opens', 'java.base/java.lang=ALL-UNNAMED'
    ]
}
```

#### Removed APIs

| API | Removed In | Replacement |
|---|---|---|
| Nashorn JS engine | Java 15 | GraalVM JS or SpEL |
| `SecurityManager` | Java 18 | OS-level security, containers |
| `sun.misc.Unsafe` (restricted) | Java 17+ | `VarHandle` API (Java 9+) |
| `java.xml.bind` (JAXB) | Java 11 | Add `jakarta.xml.bind:jakarta.xml.bind-api` explicitly |
| `java.activation` | Java 11 | Add `jakarta.activation:jakarta.activation-api` explicitly |
| `java.corba` | Java 11 | No replacement — remove CORBA usage |

#### sun.misc.Unsafe

Libraries that use `Unsafe` directly will fail. Common offenders:

- Caffeine cache < 3.0
- Netty < 4.1.70
- Kryo < 5.1
- Objenesis < 3.2

**Fix**: Upgrade the dependency. If impossible, add `--add-opens jdk.unsupported/sun.misc=ALL-UNNAMED`.

### Java 17 → 21

Fewer breaking changes. Key additions to leverage:

- **Virtual threads** — `Executors.newVirtualThreadPerTaskExecutor()`, Spring Boot 3.2+ supports `spring.threads.virtual.enabled=true`
- **Record patterns** — destructure in `instanceof` and `switch`
- **Sequenced collections** — `SequencedCollection`, `SequencedMap`
- **Pattern matching for switch** — replaces verbose `if/else instanceof` chains

---

## Common ClassNotFoundException / NoSuchMethodError Patterns

| Error | Cause | Fix |
|---|---|---|
| `ClassNotFoundException: javax.servlet.http.HttpServletRequest` | Using Spring Boot 3.x with javax imports | Change to `jakarta.servlet.*` |
| `NoSuchMethodError: ByteBuffer.flip()` | Netty version mismatch (Java 9+ covariant return) | Upgrade Netty to 4.1.100+ |
| `ClassNotFoundException: javax.xml.bind.JAXBException` | JAXB removed from JDK 11+ | Add `jakarta.xml.bind:jakarta.xml.bind-api` |
| `NoSuchMethodError: javax.annotation.PostConstruct` | javax.annotation removed from JDK 11+ | Add `jakarta.annotation:jakarta.annotation-api` |
| `IllegalAccessError: sun.misc.Unsafe` | Unsafe access blocked in Java 17+ | Upgrade library or add `--add-opens` |

---

## Dependency Conflict Detection

```bash
# Find duplicate classes across JARs (javax/jakarta mixing)
./gradlew dependencies --configuration runtimeClasspath | grep -E "javax|jakarta"

# Dependency insight for a specific library
./gradlew dependencyInsight --dependency javax.servlet

# Force resolution to jakarta
configurations.all {
    resolutionStrategy {
        force 'jakarta.servlet:jakarta.servlet-api:6.0.0'
    }
}

# Exclude javax transitives
configurations.all {
    exclude group: 'javax.servlet', module: 'servlet-api'
    exclude group: 'javax.servlet', module: 'javax.servlet-api'
}
```

---

## Spring Boot 2.x Debug Logging

For legacy projects without actuator:

```bash
# Enable auto-config report on startup
./gradlew bootRun --args='--debug'

# Or in application.properties
debug=true
logging.level.org.springframework=DEBUG

# Specific subsystems
logging.level.org.springframework.beans.factory=DEBUG     # Bean creation
logging.level.org.springframework.context=DEBUG           # Context loading
logging.level.org.springframework.web=DEBUG               # Request handling
logging.level.org.hibernate.SQL=DEBUG                     # SQL queries
logging.level.org.hibernate.type.descriptor.sql=TRACE     # SQL bind params
```

---

## Migration Checklist

### Pre-Migration
- [ ] Document current Spring Boot, Java, and dependency versions
- [ ] Run `./gradlew dependencies | grep javax` to find all javax usage
- [ ] Identify `sun.misc.Unsafe` usage in dependencies
- [ ] Check third-party library Java 17+ / Spring Boot 3.x compatibility

### Code Migration
- [ ] Replace all `javax.*` imports with `jakarta.*`
- [ ] Replace `WebSecurityConfigurerAdapter` with `SecurityFilterChain` bean
- [ ] Replace `antMatchers()` with `requestMatchers()`
- [ ] Replace `RestTemplate` with `RestClient` or `WebClient`
- [ ] Remove explicit Hibernate dialect (auto-detected in 3.x)
- [ ] Update renamed properties (`spring.resources.*` → `spring.web.resources.*`)

### Build Configuration
- [ ] Set Java 17+ in `build.gradle` (`sourceCompatibility`)
- [ ] Add `--add-opens` JVM args for tests and bootRun
- [ ] Exclude `javax.*` transitive dependencies
- [ ] Update Spring Boot Gradle plugin version

### Validation
- [ ] `./gradlew compileJava` passes with zero `javax.*` imports
- [ ] `./gradlew test` passes
- [ ] No `ClassNotFoundException` or `NoSuchMethodError` at startup
- [ ] Remove `spring-boot-properties-migrator` after migration complete
