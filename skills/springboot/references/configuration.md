# Spring Boot Configuration Reference

## @ConfigurationProperties

```java
@ConfigurationProperties(prefix = "app")
@Validated
public record AppProperties(
    @Valid SecurityProperties security,
    @Valid FeatureProperties features,
    @Valid ExternalApiProperties externalApi
) {}

public record SecurityProperties(
    @NotBlank String jwtSecret,
    @Positive long jwtExpiration,
    @Positive long refreshExpiration,
    List<String> allowedOrigins
) {}

public record FeatureProperties(
    boolean newCheckout,
    boolean recommendations,
    boolean darkMode
) {}

public record ExternalApiProperties(
    @NotNull URI baseUrl,
    String apiKey,
    @Positive int timeoutMs,
    @Positive int maxRetries
) {}
```

```java
// Register the properties class
@SpringBootApplication
@ConfigurationPropertiesScan
public class AppApplication { ... }

// Or explicitly:
@Configuration
@EnableConfigurationProperties(AppProperties.class)
public class AppConfig { }
```

---

## application.yml — Full AppProperties Example

```yaml
app:
  security:
    jwt-secret: ${JWT_SECRET}
    jwt-expiration: 86400000
    refresh-expiration: 604800000
    allowed-origins:
      - http://localhost:3000
      - https://app.example.com

  features:
    new-checkout: ${FEATURE_NEW_CHECKOUT:false}
    recommendations: true
    dark-mode: false

  external-api:
    base-url: https://api.partner.com
    api-key: ${PARTNER_API_KEY}
    timeout-ms: 5000
    max-retries: 3
```

---

## Injecting @ConfigurationProperties into Services

```java
@Service
@RequiredArgsConstructor
public class ExternalApiService {

    private final AppProperties appProperties;
    private final RestClient restClient;

    public ApiResponse call(String endpoint) {
        var apiProps = appProperties.externalApi();
        return restClient.get()
            .uri(apiProps.baseUrl().resolve(endpoint))
            .header("X-API-Key", apiProps.apiKey())
            .retrieve()
            .body(ApiResponse.class);
    }
}
```

---

## @Value and Environment

```java
@Service
public class ReportService {

    @Value("${app.reports.output-dir:/tmp/reports}")
    private Path outputDir;

    @Value("${app.reports.formats:PDF,CSV}")
    private List<String> supportedFormats;

    @Value("${spring.application.name}")
    private String appName;

    // Using Environment directly (useful when value might not be present)
    private final Environment env;

    public ReportService(Environment env) {
        this.env = env;
    }

    public boolean isDebugMode() {
        return env.getProperty("app.debug", Boolean.class, false);
    }
}
```

---

## Profile-Based Configuration

```java
// Active only for 'dev' profile
@Component
@Profile("dev")
public class DevDataSeeder implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        // seed test data
    }
}

// Active for multiple profiles
@Configuration
@Profile({"dev", "staging"})
public class DevEmailConfig {
    @Bean
    public JavaMailSender mailSender() {
        // MailHog or similar mock SMTP
        var sender = new JavaMailSenderImpl();
        sender.setHost("localhost");
        sender.setPort(1025);
        return sender;
    }
}

// Negation — active for everything except production
@Component
@Profile("!prod")
public class SwaggerAutoConfig { ... }
```

---

## Multi-Profile YAML

```yaml
# application.yml — base config (no spring.config.activate.on-profile needed here)
spring:
  application:
    name: app
  jpa:
    show-sql: false
server:
  port: 8080

---
spring:
  config:
    activate:
      on-profile: dev
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: validate
logging:
  level:
    com.example.app: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: ${DATABASE_URL}
    username: ${DB_USER}
    password: ${DB_PASS}
  jpa:
    hibernate:
      ddl-auto: validate
logging:
  level:
    root: WARN
    com.example.app: INFO
```

---

## Activating Profiles

```bash
# JVM argument
java -Dspring.profiles.active=dev -jar app.jar

# Environment variable (preferred for containers)
export SPRING_PROFILES_ACTIVE=prod
java -jar app.jar

# In tests
@SpringBootTest
@ActiveProfiles("test")
class MyTest { }
```

---

## Spring Cloud Config Client

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

```yaml
# src/main/resources/application.yml
spring:
  config:
    import: optional:configserver:${CONFIG_SERVER_URL:http://localhost:8888}
  cloud:
    config:
      fail-fast: true
      retry:
        max-attempts: 6
        initial-interval: 1000
        max-interval: 2000
      label: main
```

---

## Environment Variables — Priority Order

Spring Boot property source priority (highest to lowest):
1. Command-line arguments (`--server.port=9090`)
2. `SPRING_APPLICATION_JSON` env var
3. OS environment variables (`SERVER_PORT=9090`)
4. JVM system properties (`-Dserver.port=9090`)
5. `application-{profile}.yml`
6. `application.yml`
7. `@PropertySource` annotations

---

## Secrets — Best Practices

```yaml
# application.yml — reference env vars, never hardcode secrets
app:
  security:
    jwt-secret: ${JWT_SECRET}           # required; no default — startup fails if absent
    db-password: ${DB_PASSWORD}
    api-key: ${EXTERNAL_API_KEY:}       # optional; empty string if not set

# In production: inject via Kubernetes Secrets, AWS Secrets Manager, Azure Key Vault
```

```java
// Fail fast if required secret is missing
@ConfigurationProperties(prefix = "app.security")
@Validated
public record SecurityProperties(
    @NotBlank(message = "JWT secret must be configured (JWT_SECRET env var)")
    String jwtSecret
) {}
```

---

## Conditional Beans

```java
@Configuration
public class StorageConfig {

    @Bean
    @ConditionalOnProperty(name = "app.storage.type", havingValue = "s3")
    public StorageService s3StorageService(S3Client s3Client) {
        return new S3StorageService(s3Client);
    }

    @Bean
    @ConditionalOnProperty(name = "app.storage.type", havingValue = "local", matchIfMissing = true)
    public StorageService localStorageService(@Value("${app.storage.path:/tmp/uploads}") Path path) {
        return new LocalStorageService(path);
    }

    @Bean
    @ConditionalOnMissingBean(StorageService.class)
    public StorageService noopStorageService() {
        return new NoopStorageService();
    }
}
```

---

## Application Events

```java
@Component
@Slf4j
public class AppStartupListener {

    @EventListener(ApplicationReadyEvent.class)
    public void onReady(ApplicationReadyEvent event) {
        log.info("Application started successfully on port {}",
            event.getApplicationContext()
                .getEnvironment().getProperty("local.server.port"));
    }

    @EventListener(ApplicationFailedEvent.class)
    public void onFailure(ApplicationFailedEvent event) {
        log.error("Application failed to start", event.getException());
    }
}
```

---

## @ConfigurationProperties — Metadata for IDE Support

Create `src/main/resources/META-INF/additional-spring-configuration-metadata.json`:

```json
{
  "properties": [
    {
      "name": "app.features.new-checkout",
      "type": "java.lang.Boolean",
      "description": "Enables the redesigned checkout flow.",
      "defaultValue": false
    },
    {
      "name": "app.security.jwt-expiration",
      "type": "java.lang.Long",
      "description": "JWT access token expiration in milliseconds.",
      "defaultValue": 86400000
    }
  ]
}
```
