# Spring Boot Data Access Reference

## JPA Entity

```java
package com.example.app.entity;

import jakarta.persistence.*;
import lombok.*;
import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.jpa.domain.support.AuditingEntityListener;

import java.time.LocalDateTime;

@Entity
@Table(name = "users", indexes = {
    @Index(name = "idx_users_email", columnList = "email", unique = true)
})
@EntityListeners(AuditingEntityListener.class)
@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true, length = 255)
    private String email;

    @Column(nullable = false)
    private String password;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false, length = 20)
    private Role role;

    @Column(nullable = false)
    private boolean enabled = true;

    @CreatedDate
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @Version
    private Long version;   // optimistic locking
}

public enum Role {
    USER, ADMIN, MANAGER
}
```

---

## JPA Auditing Configuration

```java
@Configuration
@EnableJpaAuditing
public class JpaAuditingConfig {

    @Bean
    public AuditorAware<String> auditorAware() {
        return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .map(Authentication::getName);
    }
}
```

---

## Spring Data JPA Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    Optional<User> findByEmail(String email);

    boolean existsByEmail(String email);

    // Derived query method — generates JPQL automatically
    List<User> findByRoleAndEnabled(Role role, boolean enabled);

    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.name LIKE %:name% AND u.enabled = true")
    Page<User> searchByName(@Param("name") String name, Pageable pageable);

    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE created_at > :since", nativeQuery = true)
    List<User> findCreatedAfter(@Param("since") LocalDateTime since);

    // Projection — only load specific fields
    @Query("SELECT u.id AS id, u.name AS name, u.email AS email FROM User u WHERE u.role = :role")
    List<UserProjection> findProjectionsByRole(@Param("role") Role role);

    // Modifying query — must be in a transaction
    @Modifying
    @Query("UPDATE User u SET u.enabled = :enabled WHERE u.id = :id")
    int updateEnabled(@Param("id") Long id, @Param("enabled") boolean enabled);

    // Soft delete
    @Modifying
    @Query("DELETE FROM User u WHERE u.id = :id")
    void deleteById(@Param("id") Long id);
}

// Projection interface
public interface UserProjection {
    Long getId();
    String getName();
    String getEmail();
}
```

---

## Specifications (dynamic queries)

```java
public class UserSpecifications {

    public static Specification<User> hasName(String name) {
        return (root, query, cb) ->
            name == null ? null : cb.like(cb.lower(root.get("name")), "%" + name.toLowerCase() + "%");
    }

    public static Specification<User> hasRole(Role role) {
        return (root, query, cb) ->
            role == null ? null : cb.equal(root.get("role"), role);
    }

    public static Specification<User> isEnabled(Boolean enabled) {
        return (root, query, cb) ->
            enabled == null ? null : cb.equal(root.get("enabled"), enabled);
    }
}

// Repository must extend JpaSpecificationExecutor<User>
public interface UserRepository extends JpaRepository<User, Long>, JpaSpecificationExecutor<User> {}

// Usage in service
Specification<User> spec = Specification.where(UserSpecifications.hasName(name))
    .and(UserSpecifications.hasRole(role))
    .and(UserSpecifications.isEnabled(true));
Page<User> users = userRepository.findAll(spec, pageable);
```

---

## Service with @Transactional

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class UserService {

    private final UserRepository userRepository;
    private final UserMapper userMapper;
    private final PasswordEncoder passwordEncoder;

    @Transactional(readOnly = true)
    public Page<UserResponse> findAll(Pageable pageable) {
        return userRepository.findAll(pageable).map(userMapper::toResponse);
    }

    @Transactional(readOnly = true)
    public UserResponse findById(Long id) {
        return userRepository.findById(id)
            .map(userMapper::toResponse)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }

    @Transactional
    public UserResponse create(UserRequest request) {
        if (userRepository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("Email already registered: " + request.email());
        }
        var user = userMapper.toEntity(request);
        user.setPassword(passwordEncoder.encode(request.password()));
        return userMapper.toResponse(userRepository.save(user));
    }

    @Transactional
    public UserResponse update(Long id, UserRequest request) {
        var user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
        userMapper.updateEntityFromRequest(request, user);
        if (request.password() != null) {
            user.setPassword(passwordEncoder.encode(request.password()));
        }
        return userMapper.toResponse(userRepository.save(user));
    }

    @Transactional
    public void delete(Long id) {
        if (!userRepository.existsById(id)) {
            throw new ResourceNotFoundException("User", id);
        }
        userRepository.deleteById(id);
    }
}
```

---

## Flyway Migration Scripts

```sql
-- src/main/resources/db/migration/V1__init_schema.sql
CREATE TABLE users (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100)  NOT NULL,
    email       VARCHAR(255)  NOT NULL UNIQUE,
    password    VARCHAR(255)  NOT NULL,
    role        VARCHAR(20)   NOT NULL DEFAULT 'USER',
    enabled     BOOLEAN       NOT NULL DEFAULT TRUE,
    created_at  TIMESTAMP     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP     NOT NULL DEFAULT NOW(),
    version     BIGINT        NOT NULL DEFAULT 0
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_role  ON users(role);
```

```sql
-- src/main/resources/db/migration/V2__add_orders_table.sql
CREATE TABLE orders (
    id          BIGSERIAL PRIMARY KEY,
    user_id     BIGINT        NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    status      VARCHAR(20)   NOT NULL DEFAULT 'PENDING',
    total       NUMERIC(15,2) NOT NULL,
    created_at  TIMESTAMP     NOT NULL DEFAULT NOW(),
    updated_at  TIMESTAMP     NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status  ON orders(status);
```

---

## Liquibase Changeset

```yaml
# src/main/resources/db/changelog/db.changelog-master.yaml
databaseChangeLog:
  - changeSet:
      id: 001-create-users
      author: dev
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: name
                  type: VARCHAR(100)
                  constraints:
                    nullable: false
```

---

## Spring Data MongoDB

```java
@Document(collection = "products")
@Getter @Setter @Builder
public class Product {

    @Id
    private String id;

    @Indexed
    private String name;

    private String description;
    private BigDecimal price;
    private String category;

    @CreatedDate
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;
}

public interface ProductRepository extends MongoRepository<Product, String> {

    List<Product> findByCategory(String category);

    @Query("{ 'price': { $lte: ?0 }, 'category': ?1 }")
    List<Product> findByCategoryAndMaxPrice(BigDecimal maxPrice, String category);
}
```

---

## Spring Data Redis (RedisTemplate)

```java
@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
        var template = new RedisTemplate<String, Object>();
        template.setConnectionFactory(factory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}

@Repository
@RequiredArgsConstructor
public class SessionRepository {

    private final RedisTemplate<String, Object> redisTemplate;
    private static final String PREFIX = "session:";
    private static final Duration TTL = Duration.ofHours(1);

    public void save(String token, UserSession session) {
        redisTemplate.opsForValue().set(PREFIX + token, session, TTL);
    }

    public Optional<UserSession> find(String token) {
        return Optional.ofNullable(
            (UserSession) redisTemplate.opsForValue().get(PREFIX + token));
    }

    public void delete(String token) {
        redisTemplate.delete(PREFIX + token);
    }
}
```

---

## JDBC Template (without JPA)

```java
@Repository
@RequiredArgsConstructor
public class ReportRepository {

    private final NamedParameterJdbcTemplate jdbc;

    public List<ReportRow> findByDateRange(LocalDate from, LocalDate to) {
        var sql = """
            SELECT date, category, SUM(amount) AS total
            FROM transactions
            WHERE date BETWEEN :from AND :to
            GROUP BY date, category
            ORDER BY date
            """;
        var params = Map.of("from", from, "to", to);
        return jdbc.query(sql, params, (rs, row) -> new ReportRow(
            rs.getObject("date", LocalDate.class),
            rs.getString("category"),
            rs.getBigDecimal("total")
        ));
    }
}
```

---

## Pagination Best Practices

```java
// Always use PageRequest with Sort
Pageable pageable = PageRequest.of(
    page,
    size,
    Sort.by(Sort.Direction.DESC, "createdAt")
);

// Slice for cursor-based pagination (no COUNT query)
Slice<User> slice = userRepository.findAll(pageable);

// Custom pageable from request params
@GetMapping
public Page<UserResponse> findAll(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "createdAt") String sortBy,
        @RequestParam(defaultValue = "desc") String direction) {
    var sort = direction.equalsIgnoreCase("asc")
        ? Sort.by(sortBy).ascending()
        : Sort.by(sortBy).descending();
    return userService.findAll(PageRequest.of(page, size, sort));
}
```
