# Java + Spring Boot Code Examples

## Constructor Injection

```java
@Service
public class OrderService implements CreateOrderUseCase {

    private final OrderRepository orderRepository;
    private final PaymentGateway paymentGateway;

    public OrderService(OrderRepository orderRepository, PaymentGateway paymentGateway) {
        this.orderRepository = orderRepository;
        this.paymentGateway = paymentGateway;
    }
}
```

## Configuration Properties (Records)

```java
@ConfigurationProperties(prefix = "app.payment")
public record PaymentProperties(String apiKey, Duration timeout, int maxRetries) {}
```

## Config Server Placeholders

```yaml
# Served by Spring Cloud Config Server — NOT in application source
app:
  payment:
    api-key: ${PAYMENT_API_KEY}
    timeout: 30s
    max-retries: 3
  datasource:
    url: ${DATABASE_URL}
    username: ${DATABASE_USERNAME}
    password: ${DATABASE_PASSWORD}
```

## DTOs as Records

```java
public record CreateOrderRequest(String customerId, List<LineItem> items) {}
public record OrderResponse(String orderId, BigDecimal total, OrderStatus status) {}
```

## Testcontainers

```java
@Testcontainers
@SpringBootTest
class UserRepositoryIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres =
            new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
}
```

## StepVerifier

```java
@Test
void shouldReturnUser_whenIdExists() {
    StepVerifier.create(userService.findById(1L))
            .assertNext(user -> assertThat(user.name()).isEqualTo("Alice"))
            .verifyComplete();
}
```

## Error Handling

### Global Exception Handler (RFC 7807)

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setTitle("Resource Not Found");
        return problem;
    }
}
```

### Domain Exception

```java
public class ResourceNotFoundException extends DomainException {
    public ResourceNotFoundException(String resourceType, Object id) {
        super("%s not found with id: %s".formatted(resourceType, id));
    }
}
```

## WebFlux Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final FindUserUseCase findUserUseCase;

    public UserController(FindUserUseCase findUserUseCase) {
        this.findUserUseCase = findUserUseCase;
    }

    @GetMapping("/{id}")
    public Mono<UserResponse> getUser(@PathVariable Long id) {
        return findUserUseCase.findById(id)
                .map(UserResponse::from)
                .switchIfEmpty(Mono.error(new ResourceNotFoundException("User", id)));
    }

    @GetMapping
    public Flux<UserResponse> getAllUsers() {
        return findUserUseCase.findAll().map(UserResponse::from);
    }
}
```

## Reactive Composition

```java
public Mono<OrderResponse> createOrder(CreateOrderRequest request) {
    return validateOrder(request)
            .flatMap(this::processPayment)
            .flatMap(this::saveOrder)
            .map(OrderResponse::from);
}
```

## Wrapping Blocking Calls

```java
Mono.fromCallable(() -> blockingLegacyService.call(param))
        .subscribeOn(Schedulers.boundedElastic());
```

## SpringDoc OpenAPI

### Application Metadata

```java
@OpenAPIDefinition(
    info = @Info(title = "My Service API", version = "1.0.0", description = "Service description")
)
@SpringBootApplication
public class Application { }
```

### Controller Annotations

```java
@RestController
@RequestMapping("/api/users")
@Tag(name = "Users", description = "User management")
public class UserController {

    @Operation(summary = "Get user by ID")
    @ApiResponse(responseCode = "200", description = "User found")
    @ApiResponse(responseCode = "404", description = "User not found")
    @GetMapping("/{id}")
    public Mono<UserResponse> getUser(@Parameter(description = "User ID") @PathVariable Long id) {
        return findUserUseCase.findById(id).map(UserResponse::from);
    }
}
```

### Schema on DTOs

```java
@Schema(description = "Create order request")
public record CreateOrderRequest(
    @Schema(description = "Customer identifier", example = "cust-123") String customerId,
    @Schema(description = "Line items to order") List<LineItem> items
) {}
```

### Configuration

```yaml
springdoc:
  api-docs:
    path: /v3/api-docs
  swagger-ui:
    path: /swagger-ui.html
    operations-sorter: method
    tags-sorter: alpha
  show-actuator: false
  default-produces-media-type: application/json
```

## Actuator Security

### SecurityFilterChain for Actuator Endpoints

```java
/**
 * Secures actuator endpoints with dedicated credentials, separate from app auth.
 * Only /actuator/health and /actuator/info are public.
 */
@Configuration
public class ActuatorSecurityConfig {

    private final ActuatorSecurityProperties properties;

    public ActuatorSecurityConfig(ActuatorSecurityProperties properties) {
        this.properties = properties;
    }

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorFilterChain(HttpSecurity http) throws Exception {
        http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/actuator/**").authenticated()
            )
            .httpBasic(Customizer.withDefaults());
        return http.build();
    }

    @Bean
    public InMemoryUserDetailsManager actuatorUserDetailsManager(PasswordEncoder encoder) {
        UserDetails actuatorUser = User.builder()
            .username(properties.username())
            .password(encoder.encode(properties.password()))
            .roles("ACTUATOR")
            .build();
        return new InMemoryUserDetailsManager(actuatorUser);
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

```java
@ConfigurationProperties(prefix = "management.auth")
public record ActuatorSecurityProperties(String username, String password) {}
```

```yaml
# Served by Spring Cloud Config Server — NOT in application source
management:
  auth:
    username: ${ACTUATOR_USERNAME}
    password: ${ACTUATOR_PASSWORD}
```

### Actuator Sanitization (application.yml)

```yaml
management:
  endpoints:
    web:
      exposure:
        include: env,health,info,beans,conditions,mappings,refresh
  endpoint:
    env:
      show-values: WHEN_AUTHORIZED
    health:
      show-details: WHEN_AUTHORIZED
  server:
    port: 9090   # Optional: bind actuator to internal-only port
```
