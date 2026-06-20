---
name: springboot
description: >
  Generate, scaffold, and implement Spring Boot 3.x application code covering the full
  development lifecycle. Covers web layer (REST controllers, @RestController, @GetMapping,
  @PostMapping, @PutMapping, @DeleteMapping, @PatchMapping, @RequestBody, @PathVariable,
  @RequestParam, ResponseEntity, @Valid, @Validated, GlobalExceptionHandler,
  @ControllerAdvice, @ExceptionHandler, ProblemDetail, WebFlux reactive controllers,
  RouterFunction, ServerRequest), data access (Spring Data JPA, @Entity, @Repository,
  JpaRepository, CrudRepository, JPQL, native queries, @Transactional, @Query, Pageable,
  Page, Slice, Specification, database migration with Flyway, Liquibase, Spring Data
  MongoDB, Spring Data Redis, JDBC Template, named parameters), security (Spring Security 6,
  SecurityFilterChain, JWT authentication, JwtAuthenticationFilter, UserDetailsService,
  PasswordEncoder, BCryptPasswordEncoder, OAuth2, OAuth2ResourceServer, method security,
  @PreAuthorize, @PostAuthorize, @Secured, CORS, CSRF, basic auth, form login), testing
  (JUnit 5, @Test, @SpringBootTest, @WebMvcTest, MockMvc, @AutoConfigureMockMvc, @DataJpaTest,
  @MockBean, @SpyBean, Mockito, mock, verify, @ExtendWith(MockitoExtension), given/when/then,
  Testcontainers, @Container, @DynamicPropertySource, @TestConfiguration, @TestPropertySource,
  WebTestClient, @WebFluxTest, parameterized tests, @ParameterizedTest, @ValueSource,
  @CsvSource, AssertJ, assertThat), messaging (Spring Kafka, @KafkaListener, KafkaTemplate,
  ProducerRecord, ConsumerRecord, @EnableKafka, DeadLetterPublishingRecoverer, Spring AMQP,
  RabbitMQ, @RabbitListener, RabbitTemplate, @EnableRabbit, MessageConverter), caching
  (Spring Cache, @EnableCaching, @Cacheable, @CacheEvict, @CachePut, Redis cache manager,
  RedisCacheConfiguration, Caffeine cache, CaffeineSpec), observability (Spring Boot Actuator,
  /actuator/health, /actuator/metrics, /actuator/info, custom HealthIndicator,
  Micrometer, Counter, Timer, Gauge, MeterRegistry, Micrometer Tracing, @Observed,
  ObservationRegistry, distributed tracing, Zipkin, Jaeger), configuration
  (@ConfigurationProperties, @EnableConfigurationProperties, @Value, Environment,
  application.properties, application.yml, Spring profiles, @Profile, @ActiveProfiles,
  multi-profile YAML, Spring Cloud Config), project structure (Maven pom.xml, Gradle
  build.gradle, Spring Initializr, multi-module Maven/Gradle, application.yml patterns,
  BOM, dependency management), build and deployment (mvn package, gradle build,
  Dockerfile, multi-stage Docker build, docker-compose, GitHub Actions workflow,
  JVM tuning, GraalVM native image). Use when asked to scaffold, generate, create, write,
  implement, or fix Spring Boot code for any of: REST API, controller, service, repository,
  entity, DTO, mapper, validator, filter, interceptor, aspect, AOP, exception handler,
  security config, JWT, OAuth2, test, unit test, integration test, MockMvc, Testcontainers,
  Kafka, RabbitMQ, cache, Redis, Caffeine, actuator, health check, metrics, tracing,
  configuration properties, profile, Dockerfile, CI/CD pipeline, Maven, Gradle.
allowed-tools: Read Grep Glob Write Edit Bash
argument-hint: "[area] [feature-name] -- e.g. web UserController or testing UserServiceTest or security JwtFilter"
---

## Invocation and Argument Parsing

When invoked, parse `$ARGUMENTS` for:
- **First argument** (optional): area — one of:
  `web`, `rest`, `controller`, `mvc`, `webflux`, `reactive`,
  `data`, `jpa`, `repository`, `entity`, `database`, `migration`, `flyway`, `liquibase`,
  `mongo`, `mongodb`, `redis`, `jdbc`,
  `security`, `jwt`, `oauth2`, `auth`, `authentication`, `authorization`,
  `testing`, `test`, `unittest`, `integration`, `mockito`, `testcontainers`, `webmvctest`, `datajpatest`,
  `messaging`, `kafka`, `rabbitmq`, `amqp`,
  `caching`, `cache`, `caffeine`,
  `actuator`, `observability`, `metrics`, `tracing`, `health`,
  `config`, `configuration`, `properties`, `profiles`,
  `build`, `docker`, `ci`, `gradle`, `maven`,
  `project`, `structure`, `scaffold`, `initializr`
- **Second argument** (optional): feature or class name (e.g., `UserController`, `OrderService`, `ProductRepository`)

If either is missing, infer from the user's description. Ask only if both are completely absent:
> "Which Spring Boot area? And what should the class/feature be named?"

---

## Generation Procedure

Follow these steps for every request.

### Step 1 — Identify the area and load the reference file

Map the requested area to its reference file:

| Area / topic | Reference file |
|---|---|
| REST controllers, request mappings, exception handling, validation, WebFlux | `references/web-layer.md` |
| JPA entities, repositories, transactions, queries, migrations, MongoDB, JDBC | `references/data-access.md` |
| Spring Security, JWT, OAuth2, method security, CORS, CSRF | `references/security.md` |
| JUnit 5, Mockito, MockMvc, Testcontainers, test slices, WebTestClient | `references/testing.md` |
| Kafka, RabbitMQ, Spring Messaging, dead-letter queues | `references/messaging.md` |
| @Cacheable, Redis cache, Caffeine, cache eviction | `references/caching.md` |
| Actuator, Micrometer, custom health indicators, distributed tracing | `references/actuator.md` |
| @ConfigurationProperties, profiles, externalized config, Spring Cloud Config | `references/configuration.md` |
| Maven, Gradle, Dockerfile, docker-compose, GitHub Actions | `references/build-deploy.md` |
| Project setup, pom.xml, application.yml, directory layout, multi-module | `references/project-structure.md` |

**Critical**: Use `Glob` to find the absolute path first, then `Read` the result:

```
Glob pattern: **/skills/springboot/references/<filename>
```

If Glob returns no results:
> "Could not locate the skill reference files. Verify the repo is added via `--add-dir /path/to/development-claude-skills`."

For full-stack features, load multiple reference files (e.g., web + data + testing).

### Step 2 — Understand the project context

Before generating code, use `Grep` and `Glob` to inspect the user's project:
- Check `pom.xml` or `build.gradle` for Spring Boot version, existing dependencies, and Java version
- Check `src/main/java` package structure to match naming conventions
- Check `src/main/resources/application.yml` or `.properties` for existing config keys
- Check if Jakarta EE imports are used (Spring Boot 3.x) vs javax (Spring Boot 2.x)

Adapt all generated code to match the existing project's conventions.

### Step 3 — Generate the code

Apply patterns from the reference file. Rules:
- Use constructor injection, never `@Autowired` on fields
- Use `var` for local variables where the type is obvious (Java 17+)
- Use records for DTOs and value objects where appropriate (Java 16+)
- Prefer `Optional` return types from repositories, never return `null`
- Follow the package layout: `controller`, `service`, `repository`, `entity`, `dto`, `exception`, `config`, `security`
- Spring Boot 3.x uses `jakarta.*` imports, not `javax.*`
- Always generate a corresponding test class alongside production code

### Step 4 — Generate the test

Always include a test class unless the user explicitly says not to:
- Unit test for service-layer classes (Mockito, no Spring context)
- `@WebMvcTest` for controllers (MockMvc)
- `@DataJpaTest` for repositories (H2 or Testcontainers)
- `@SpringBootTest` only for full integration tests
- Load `references/testing.md` for test patterns

### Step 5 — Summarise

After writing all files, print:
- Files created and their paths
- Key dependencies to add to `pom.xml` / `build.gradle` (if not already present)
- Any configuration properties to add to `application.yml`
- Next steps (e.g., "run `mvn test` to verify", "update SecurityFilterChain to permit the new endpoint")

---

## Package and Naming Conventions

```
com.<company>.<app>/
├── <AppName>Application.java         # @SpringBootApplication entry point
├── controller/                        # @RestController classes
├── service/                           # @Service interfaces + implementations
├── repository/                        # @Repository / Spring Data interfaces
├── entity/                            # @Entity JPA classes
├── dto/                               # Records or POJOs for request/response
├── exception/                         # Custom exception classes
├── config/                            # @Configuration classes
├── security/                          # Security filters, JWT utils, UserDetails
├── mapper/                            # MapStruct or manual mapper classes
└── util/                              # Utility/helper classes
```

**Naming rules:**
- Controllers: `<Feature>Controller`
- Services: `<Feature>Service` (interface) + `<Feature>ServiceImpl` (implementation)
- Repositories: `<Feature>Repository`
- Entities: singular noun (`User`, `Order`, `Product`)
- DTOs: `<Feature>Request`, `<Feature>Response`, `<Feature>Dto`
- Tests: `<ClassName>Test` (unit) or `<ClassName>IT` (integration)

---

## Spring Boot 3.x Migration Notes

| Spring Boot 2.x | Spring Boot 3.x |
|---|---|
| `javax.persistence.*` | `jakarta.persistence.*` |
| `javax.validation.*` | `jakarta.validation.*` |
| `javax.servlet.*` | `jakarta.servlet.*` |
| `WebSecurityConfigurerAdapter` | `SecurityFilterChain` bean |
| `spring.security.oauth2.resourceserver.jwt.jwk-set-uri` | same (unchanged) |
| `spring-boot-starter-validation` | same (unchanged) |

Always generate code using **Spring Boot 3.x** (`jakarta.*`) unless the project's `pom.xml` shows version `2.x`.

---

## Reference File Routing

| User says | Load these reference files |
|---|---|
| "REST controller", "endpoint", "API", "request mapping", "GET/POST/PUT/DELETE", "@RestController", "ResponseEntity", "validation", "@Valid", "exception handler", "WebFlux", "reactive" | `references/web-layer.md` |
| "JPA", "entity", "repository", "Spring Data", "@Entity", "@Repository", "JPQL", "query", "transaction", "@Transactional", "Flyway", "Liquibase", "migration", "MongoDB", "Redis data", "JDBC" | `references/data-access.md` |
| "security", "JWT", "token", "authentication", "authorization", "OAuth2", "Spring Security", "UserDetails", "PasswordEncoder", "@PreAuthorize", "CORS", "CSRF", "login", "filter" | `references/security.md` |
| "test", "JUnit", "Mockito", "MockMvc", "@SpringBootTest", "@WebMvcTest", "@DataJpaTest", "Testcontainers", "mock", "verify", "assert", "WebTestClient" | `references/testing.md` |
| "Kafka", "consumer", "producer", "@KafkaListener", "KafkaTemplate", "RabbitMQ", "@RabbitListener", "RabbitTemplate", "messaging", "queue", "topic", "dead letter" | `references/messaging.md` |
| "cache", "@Cacheable", "@CacheEvict", "Redis cache", "Caffeine", "cache manager", "cache eviction", "cache put" | `references/caching.md` |
| "actuator", "health", "metrics", "Micrometer", "tracing", "Zipkin", "Jaeger", "counter", "timer", "gauge", "@Observed", "health indicator" | `references/actuator.md` |
| "@ConfigurationProperties", "application.yml", "application.properties", "profiles", "@Profile", "externalized config", "Spring Cloud Config", "@Value" | `references/configuration.md` |
| "Maven", "Gradle", "pom.xml", "build.gradle", "Dockerfile", "docker-compose", "GitHub Actions", "CI/CD", "native image", "GraalVM", "jar", "build" | `references/build-deploy.md` |
| "project setup", "scaffold", "initializr", "new project", "pom.xml setup", "directory structure", "multi-module" | `references/project-structure.md` |
| "full stack", "complete feature", "end to end", "everything" | all reference files |

When in doubt, load the file — it is cheap and prevents hallucinated patterns.

---

## Error Handling

- If the user's project uses Spring Boot 2.x, generate `javax.*` imports and `WebSecurityConfigurerAdapter` patterns.
- If a reference file cannot be located by Glob, report the attempted pattern and ask the user to verify the `--add-dir` path.
- Never invent annotations or class names not present in the reference templates. Add `// TODO: verify` if uncertain.
- If the user asks for Kotlin, note that reference files are Java-based and translate the patterns to idiomatic Kotlin.
