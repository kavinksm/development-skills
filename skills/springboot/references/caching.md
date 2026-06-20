# Spring Boot Caching Reference

## Enable Caching

```java
@SpringBootApplication
@EnableCaching
public class AppApplication { ... }
```

---

## @Cacheable, @CachePut, @CacheEvict

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class ProductService {

    private final ProductRepository productRepository;
    private final ProductMapper productMapper;

    // Cache result; key = product ID
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public ProductResponse findById(Long id) {
        log.debug("Fetching product from DB: {}", id);
        return productRepository.findById(id)
            .map(productMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
    }

    // Cache all products with a fixed key
    @Cacheable(value = "products", key = "'all'")
    public List<ProductResponse> findAll() {
        return productMapper.toResponseList(productRepository.findAll());
    }

    // Update cache after saving
    @CachePut(value = "products", key = "#result.id()")
    @Transactional
    public ProductResponse create(ProductRequest request) {
        var product = productMapper.toEntity(request);
        return productMapper.toResponse(productRepository.save(product));
    }

    // Update cache on update
    @CachePut(value = "products", key = "#id")
    @Transactional
    public ProductResponse update(Long id, ProductRequest request) {
        var product = productRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Product", id));
        productMapper.updateEntityFromRequest(request, product);
        return productMapper.toResponse(productRepository.save(product));
    }

    // Evict single entry on delete
    @CacheEvict(value = "products", key = "#id")
    @Transactional
    public void delete(Long id) {
        productRepository.deleteById(id);
    }

    // Evict entire cache
    @CacheEvict(value = "products", allEntries = true)
    public void clearCache() {
        log.info("Product cache cleared");
    }

    // Multiple cache operations
    @Caching(evict = {
        @CacheEvict(value = "products", key = "#id"),
        @CacheEvict(value = "productsByCategory", allEntries = true)
    })
    @Transactional
    public void deleteWithCategories(Long id) {
        productRepository.deleteById(id);
    }
}
```

---

## Redis Cache Manager

```java
@Configuration
public class RedisCacheConfig {

    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory,
                                           ObjectMapper objectMapper) {
        var jsonSerializer = new GenericJackson2JsonRedisSerializer(objectMapper);

        var defaultConfig = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(30))
            .serializeKeysWith(RedisSerializationContext.SerializationPair
                .fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair
                .fromSerializer(jsonSerializer))
            .disableCachingNullValues();

        // Per-cache TTL configuration
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products",        defaultConfig.entryTtl(Duration.ofMinutes(60)),
            "users",           defaultConfig.entryTtl(Duration.ofMinutes(15)),
            "sessions",        defaultConfig.entryTtl(Duration.ofHours(24)),
            "configurations",  defaultConfig.entryTtl(Duration.ofHours(12))
        );

        return RedisCacheManager.builder(connectionFactory)
            .cacheDefaults(defaultConfig)
            .withInitialCacheConfigurations(cacheConfigs)
            .transactionAware()
            .build();
    }
}
```

---

## Caffeine Cache (in-process)

```java
@Configuration
@EnableCaching
public class CaffeineCacheConfig {

    @Bean
    public CacheManager cacheManager() {
        var caffeineSpec = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(Duration.ofMinutes(30))
            .expireAfterAccess(Duration.ofMinutes(10))
            .recordStats();

        var manager = new CaffeineCacheManager();
        manager.setCaffeine(caffeineSpec);
        manager.setCacheNames(List.of("products", "users", "configurations"));
        return manager;
    }
}
```

```yaml
# application.yml — Caffeine via properties
spring:
  cache:
    type: caffeine
    caffeine:
      spec: maximumSize=1000,expireAfterWrite=30m,recordStats
    cache-names:
      - products
      - users
      - configurations
```

---

## Conditional Caching

```java
// Only cache if size > 0 and not empty
@Cacheable(value = "search-results",
           key = "#query + '_' + #page",
           condition = "#query.length() > 2",
           unless = "#result.isEmpty()")
public List<ProductResponse> search(String query, int page) { ... }

// SpEL for complex cache keys
@Cacheable(value = "user-products",
           key = "#root.target.class.simpleName + ':' + #userId + ':' + #category")
public List<ProductResponse> findByUserAndCategory(Long userId, String category) { ... }
```

---

## Cache Warming

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class CacheWarmupService implements ApplicationRunner {

    private final ProductService productService;
    private final CacheManager cacheManager;

    @Override
    public void run(ApplicationArguments args) {
        log.info("Warming up product cache...");
        productService.findAll(); // populates 'products:all'
        log.info("Cache warm-up complete");
    }

    @Scheduled(cron = "0 0 * * * *")  // every hour
    @CacheEvict(value = "products", allEntries = true)
    public void refreshProductCache() {
        productService.findAll();
        log.info("Product cache refreshed");
    }
}
```

---

## Cache Metrics (Caffeine stats exposed to Micrometer)

```java
@Configuration
public class CacheMetricsConfig {

    @Bean
    public CacheMetricsRegistrar cacheMetricsRegistrar(
            Collection<CacheManager> cacheManagers,
            MeterRegistry meterRegistry) {
        return new CacheMetricsRegistrar(meterRegistry, cacheManagers);
    }
}
```

---

## Custom Cache Key Generator

```java
@Component("customKeyGenerator")
public class CustomCacheKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        return target.getClass().getSimpleName()
            + ":" + method.getName()
            + ":" + Arrays.stream(params)
                .map(Object::toString)
                .collect(Collectors.joining(":"));
    }
}

// Usage:
@Cacheable(value = "products", keyGenerator = "customKeyGenerator")
public ProductResponse findWithCustomKey(Long id, String locale) { ... }
```

---

## Cache Testing

```java
@SpringBootTest
@ActiveProfiles("test")
class ProductServiceCacheTest {

    @Autowired
    private ProductService productService;

    @Autowired
    private CacheManager cacheManager;

    @MockBean
    private ProductRepository productRepository;

    @BeforeEach
    void clearCaches() {
        cacheManager.getCacheNames().forEach(name ->
            Objects.requireNonNull(cacheManager.getCache(name)).clear());
    }

    @Test
    void findById_SecondCall_HitsCache() {
        var product = buildProduct(1L, "Widget");
        given(productRepository.findById(1L)).willReturn(Optional.of(product));

        productService.findById(1L); // cache miss — hits DB
        productService.findById(1L); // cache hit — should NOT hit DB again

        then(productRepository).should(times(1)).findById(1L);

        var cache = cacheManager.getCache("products");
        assertThat(cache).isNotNull();
        assertThat(cache.get(1L)).isNotNull();
    }

    @Test
    void delete_EvictsFromCache() {
        var product = buildProduct(1L, "Widget");
        given(productRepository.findById(1L)).willReturn(Optional.of(product));

        productService.findById(1L); // populate cache
        productService.delete(1L);   // should evict

        var cache = cacheManager.getCache("products");
        assertThat(cache.get(1L)).isNull();
    }
}
```

---

## Cache Properties

```yaml
spring:
  cache:
    type: redis   # redis | caffeine | simple | none
  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:6379}
      password: ${REDIS_PASSWORD:}
      timeout: 2000ms
      lettuce:
        pool:
          max-active: 8
          max-idle: 4
          min-idle: 1
```
