# Spring Boot Actuator & Observability Reference

## Actuator Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
<!-- For distributed tracing -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-zipkin</artifactId>
</dependency>
```

---

## Actuator Configuration

```yaml
management:
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health,info,metrics,prometheus,loggers,env,beans,caches
  endpoint:
    health:
      show-details: when-authorized
      show-components: when-authorized
      probes:
        enabled: true           # enables /actuator/health/liveness and /actuator/health/readiness
    loggers:
      enabled: true
    env:
      show-values: when-authorized
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
    db:
      enabled: true
    redis:
      enabled: true
    kafka:
      enabled: true
  tracing:
    sampling:
      probability: 1.0          # 1.0 = 100% sampling; use 0.1 in production
  metrics:
    distribution:
      percentiles-histogram:
        "[http.server.requests]": true
      slo:
        "[http.server.requests]": 50ms,100ms,200ms,500ms
    tags:
      application: ${spring.application.name}
      environment: ${spring.profiles.active:default}

info:
  app:
    name: ${spring.application.name}
    version: @project.version@
    description: @project.description@
  java:
    version: ${java.version}
```

---

## Custom Health Indicator

```java
@Component
@RequiredArgsConstructor
public class ExternalApiHealthIndicator implements HealthIndicator {

    private final ExternalApiClient apiClient;

    @Override
    public Health health() {
        try {
            var response = apiClient.ping();
            if (response.isSuccess()) {
                return Health.up()
                    .withDetail("url", apiClient.getBaseUrl())
                    .withDetail("responseTime", response.getElapsedMs() + "ms")
                    .build();
            }
            return Health.down()
                .withDetail("url", apiClient.getBaseUrl())
                .withDetail("statusCode", response.getStatusCode())
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("url", apiClient.getBaseUrl())
                .withException(e)
                .build();
        }
    }
}

// Reactive health indicator
@Component
@RequiredArgsConstructor
public class DatabaseReactiveHealthIndicator implements ReactiveHealthIndicator {

    private final R2dbcEntityTemplate template;

    @Override
    public Mono<Health> health() {
        return template.getDatabaseClient()
            .sql("SELECT 1")
            .fetch().one()
            .map(r -> Health.up().build())
            .onErrorResume(ex -> Mono.just(Health.down(ex).build()));
    }
}
```

---

## Micrometer — Counter, Timer, Gauge

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    private final OrderRepository orderRepository;
    private final MeterRegistry meterRegistry;

    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTimer;

    // Constructor-based metric initialization
    public OrderService(OrderRepository orderRepository, MeterRegistry meterRegistry) {
        this.orderRepository = orderRepository;
        this.meterRegistry = meterRegistry;

        this.ordersCreated = Counter.builder("orders.created.total")
            .description("Total number of orders created")
            .tag("app", "order-service")
            .register(meterRegistry);

        this.ordersFailed = Counter.builder("orders.failed.total")
            .description("Total number of failed orders")
            .register(meterRegistry);

        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Time taken to process an order")
            .publishPercentiles(0.5, 0.90, 0.99)
            .publishPercentileHistogram()
            .register(meterRegistry);

        // Gauge for in-flight orders
        Gauge.builder("orders.pending.count", orderRepository, OrderRepository::countPending)
            .description("Number of orders pending processing")
            .register(meterRegistry);
    }

    public OrderResponse createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            try {
                var order = processOrder(request);
                ordersCreated.increment();
                return order;
            } catch (Exception e) {
                ordersFailed.increment(Tags.of("reason", e.getClass().getSimpleName()));
                throw e;
            }
        });
    }
}
```

---

## @Observed (automatic span + metrics)

```java
@Service
@RequiredArgsConstructor
public class PaymentService {

    @Observed(
        name = "payment.process",
        contextualName = "process-payment",
        lowCardinalityKeyValues = {"payment.type", "CARD"}
    )
    public PaymentResult processPayment(PaymentRequest request) {
        // Micrometer Tracing automatically creates a span and records timer metrics
        return doProcessPayment(request);
    }
}

// Enable @Observed via AOP
@Configuration
@EnableAspectJAutoProxy
public class ObservationConfig {

    @Bean
    public ObservedAspect observedAspect(ObservationRegistry registry) {
        return new ObservedAspect(registry);
    }
}
```

---

## Custom Observation (manual spans)

```java
@Service
@RequiredArgsConstructor
public class NotificationService {

    private final ObservationRegistry observationRegistry;
    private final EmailClient emailClient;

    public void sendWelcomeEmail(String email) {
        Observation.createNotStarted("email.send", observationRegistry)
            .lowCardinalityKeyValue("email.type", "welcome")
            .highCardinalityKeyValue("email.recipient", email)
            .observe(() -> emailClient.send(email, "Welcome!"));
    }
}
```

---

## Scheduled Metrics Collection

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class SystemMetricsCollector {

    private final MeterRegistry meterRegistry;
    private final UserRepository userRepository;
    private final OrderRepository orderRepository;

    @Scheduled(fixedDelay = 60_000)
    public void collectMetrics() {
        meterRegistry.gauge("users.active.count", userRepository.countByEnabled(true));
        meterRegistry.gauge("orders.pending.count", orderRepository.countByStatus(OrderStatus.PENDING));
    }
}
```

---

## Custom Info Contributor

```java
@Component
public class FeatureFlagsInfoContributor implements InfoContributor {

    private final FeatureFlagService featureFlagService;

    @Override
    public void contribute(Info.Builder builder) {
        builder.withDetail("features", Map.of(
            "newCheckout", featureFlagService.isEnabled("new-checkout"),
            "recommendations", featureFlagService.isEnabled("recommendations")
        ));
    }
}
```

---

## Distributed Tracing — Zipkin / OTLP

```yaml
management:
  zipkin:
    tracing:
      endpoint: ${ZIPKIN_URL:http://localhost:9411}/api/v2/spans
  tracing:
    sampling:
      probability: 0.1  # sample 10% in production

# For OpenTelemetry Collector (OTLP)
management:
  otlp:
    tracing:
      endpoint: ${OTEL_EXPORTER_OTLP_ENDPOINT:http://localhost:4318}/v1/traces
```

---

## Structured Logging with Trace Context

```xml
<!-- logback-spring.xml -->
<configuration>
    <include resource="org/springframework/boot/logging/logback/defaults.xml"/>
    <springProperty scope="context" name="appName" source="spring.application.name"/>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"app":"${appName}"}</customFields>
            <includeMdcKeyName>traceId</includeMdcKeyName>
            <includeMdcKeyName>spanId</includeMdcKeyName>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

---

## Actuator Security

```java
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    @Order(1)
    public SecurityFilterChain actuatorSecurityChain(HttpSecurity http) throws Exception {
        return http
            .securityMatcher(EndpointRequest.toAnyEndpoint())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers(EndpointRequest.to(HealthEndpoint.class, InfoEndpoint.class)).permitAll()
                .requestMatchers(EndpointRequest.toAnyEndpoint()).hasRole("ACTUATOR_ADMIN")
                .anyRequest().denyAll())
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

---

## Custom Actuator Endpoint

```java
@Component
@Endpoint(id = "feature-flags")
@RequiredArgsConstructor
public class FeatureFlagEndpoint {

    private final FeatureFlagService featureFlagService;

    @ReadOperation
    public Map<String, Boolean> getFlags() {
        return featureFlagService.getAllFlags();
    }

    @WriteOperation
    public void setFlag(@Selector String flagName, boolean enabled) {
        featureFlagService.setFlag(flagName, enabled);
    }

    @DeleteOperation
    public void resetFlag(@Selector String flagName) {
        featureFlagService.reset(flagName);
    }
}
// Accessible at: GET /actuator/feature-flags
```
