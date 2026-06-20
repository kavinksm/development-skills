# Spring Boot Testing Reference

## Unit Test — Service Layer (Mockito)

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private UserMapper userMapper;

    @Mock
    private PasswordEncoder passwordEncoder;

    @InjectMocks
    private UserService userService;

    private User user;
    private UserRequest userRequest;
    private UserResponse userResponse;

    @BeforeEach
    void setUp() {
        user = User.builder()
            .id(1L)
            .name("John Doe")
            .email("john@example.com")
            .password("encoded-password")
            .role(Role.USER)
            .enabled(true)
            .build();

        userRequest = new UserRequest("John Doe", "john@example.com", "password123", "USER");
        userResponse = new UserResponse(1L, "John Doe", "john@example.com", "USER", LocalDateTime.now());
    }

    @Test
    void findById_WhenExists_ReturnsResponse() {
        given(userRepository.findById(1L)).willReturn(Optional.of(user));
        given(userMapper.toResponse(user)).willReturn(userResponse);

        var result = userService.findById(1L);

        assertThat(result).isEqualTo(userResponse);
        then(userRepository).should().findById(1L);
    }

    @Test
    void findById_WhenNotFound_ThrowsNotFoundException() {
        given(userRepository.findById(99L)).willReturn(Optional.empty());

        assertThatThrownBy(() -> userService.findById(99L))
            .isInstanceOf(ResourceNotFoundException.class)
            .hasMessageContaining("99");
    }

    @Test
    void create_WhenEmailTaken_ThrowsDuplicateException() {
        given(userRepository.existsByEmail("john@example.com")).willReturn(true);

        assertThatThrownBy(() -> userService.create(userRequest))
            .isInstanceOf(DuplicateResourceException.class);

        then(userRepository).should(never()).save(any());
    }

    @Test
    void create_WhenValid_SavesAndReturns() {
        given(userRepository.existsByEmail(anyString())).willReturn(false);
        given(userMapper.toEntity(userRequest)).willReturn(user);
        given(passwordEncoder.encode("password123")).willReturn("encoded-password");
        given(userRepository.save(user)).willReturn(user);
        given(userMapper.toResponse(user)).willReturn(userResponse);

        var result = userService.create(userRequest);

        assertThat(result).isEqualTo(userResponse);
        then(passwordEncoder).should().encode("password123");
        then(userRepository).should().save(user);
    }

    // Parameterized test
    @ParameterizedTest
    @ValueSource(strings = {"USER", "ADMIN", "MANAGER"})
    void findByRole_ReturnsMatchingUsers(String role) {
        given(userRepository.findByRoleAndEnabled(Role.valueOf(role), true))
            .willReturn(List.of(user));
        given(userMapper.toResponse(user)).willReturn(userResponse);

        var result = userService.findByRole(Role.valueOf(role));

        assertThat(result).hasSize(1);
    }

    @ParameterizedTest
    @CsvSource({
        "john@example.com, John, true",
        "admin@example.com, Admin, true",
        "unknown@example.com, '', false"
    })
    void existsByEmail(String email, String name, boolean expected) {
        given(userRepository.existsByEmail(email)).willReturn(expected);
        assertThat(userService.existsByEmail(email)).isEqualTo(expected);
    }
}
```

---

## Controller Test — @WebMvcTest

```java
@WebMvcTest(UserController.class)
@Import(SecurityConfig.class)
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private UserService userService;

    @Autowired
    private ObjectMapper objectMapper;

    private UserResponse userResponse;

    @BeforeEach
    void setUp() {
        userResponse = new UserResponse(1L, "John Doe", "john@example.com", "USER", LocalDateTime.now());
    }

    @Test
    @WithMockUser
    void findById_Returns200() throws Exception {
        given(userService.findById(1L)).willReturn(userResponse);

        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            .andExpect(jsonPath("$.id").value(1L))
            .andExpect(jsonPath("$.email").value("john@example.com"))
            .andExpect(jsonPath("$.name").value("John Doe"));
    }

    @Test
    @WithMockUser
    void findById_WhenNotFound_Returns404() throws Exception {
        given(userService.findById(99L)).willThrow(new ResourceNotFoundException("User", 99L));

        mockMvc.perform(get("/api/v1/users/99"))
            .andExpect(status().isNotFound());
    }

    @Test
    @WithMockUser
    void create_WithValidRequest_Returns201() throws Exception {
        var request = new UserRequest("John Doe", "john@example.com", "password123", "USER");
        given(userService.create(any())).willReturn(userResponse);

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(1L));
    }

    @Test
    @WithMockUser
    void create_WithInvalidEmail_Returns400() throws Exception {
        var request = new UserRequest("John Doe", "not-an-email", "password123", "USER");

        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.errors.email").exists());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void delete_AsAdmin_Returns204() throws Exception {
        willDoNothing().given(userService).delete(1L);

        mockMvc.perform(delete("/api/v1/users/1"))
            .andExpect(status().isNoContent());
    }

    @Test
    void unauthenticated_Returns401() throws Exception {
        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isUnauthorized());
    }
}
```

---

## Repository Test — @DataJpaTest

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@Testcontainers
class UserRepositoryTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @BeforeEach
    void setUp() {
        var user = User.builder()
            .name("John Doe")
            .email("john@example.com")
            .password("hash")
            .role(Role.USER)
            .enabled(true)
            .build();
        entityManager.persist(user);
        entityManager.flush();
    }

    @Test
    void findByEmail_WhenExists_ReturnsUser() {
        var result = userRepository.findByEmail("john@example.com");

        assertThat(result).isPresent();
        assertThat(result.get().getName()).isEqualTo("John Doe");
    }

    @Test
    void existsByEmail_WhenExists_ReturnsTrue() {
        assertThat(userRepository.existsByEmail("john@example.com")).isTrue();
        assertThat(userRepository.existsByEmail("unknown@example.com")).isFalse();
    }

    @Test
    void searchByName_WithPartialMatch_ReturnResults() {
        var pageable = PageRequest.of(0, 10);
        var result = userRepository.searchByName("john", pageable);

        assertThat(result.getContent()).hasSize(1);
        assertThat(result.getContent().get(0).getEmail()).isEqualTo("john@example.com");
    }
}
```

---

## Integration Test — @SpringBootTest

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
@ActiveProfiles("test")
class UserIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Autowired
    private TestRestTemplate restTemplate;

    @Autowired
    private UserRepository userRepository;

    @AfterEach
    void cleanup() {
        userRepository.deleteAll();
    }

    @Test
    void createAndRetrieveUser() {
        var request = new UserRequest("Jane Doe", "jane@example.com", "password123", "USER");

        var createResponse = restTemplate.postForEntity(
            "/api/v1/users", request, UserResponse.class);

        assertThat(createResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(createResponse.getBody()).isNotNull();
        assertThat(createResponse.getBody().email()).isEqualTo("jane@example.com");

        var id = createResponse.getBody().id();
        var getResponse = restTemplate.getForEntity(
            "/api/v1/users/" + id, UserResponse.class);

        assertThat(getResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
    }
}
```

---

## WebTestClient (WebFlux)

```java
@WebFluxTest(UserReactiveController.class)
class UserReactiveControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @MockBean
    private UserReactiveService userService;

    @Test
    void findAll_ReturnsFlux() {
        var responses = List.of(
            new UserResponse(1L, "Alice", "alice@example.com", "USER", LocalDateTime.now()),
            new UserResponse(2L, "Bob", "bob@example.com", "ADMIN", LocalDateTime.now())
        );
        given(userService.findAll()).willReturn(Flux.fromIterable(responses));

        webTestClient.get().uri("/api/v1/users")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectBodyList(UserResponse.class)
            .hasSize(2);
    }

    @Test
    void create_WithValidBody_Returns201() {
        var request = new UserRequest("Alice", "alice@example.com", "password123", "USER");
        var response = new UserResponse(1L, "Alice", "alice@example.com", "USER", LocalDateTime.now());
        given(userService.create(any())).willReturn(Mono.just(response));

        webTestClient.post().uri("/api/v1/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(request)
            .exchange()
            .expectStatus().isCreated()
            .expectBody(UserResponse.class)
            .value(r -> assertThat(r.email()).isEqualTo("alice@example.com"));
    }
}
```

---

## TestConfiguration — Shared Beans

```java
@TestConfiguration
public class TestSecurityConfig {

    @Bean
    @Primary
    public UserDetailsService testUserDetailsService() {
        var user = User.withDefaultPasswordEncoder()
            .username("test@example.com")
            .password("password")
            .roles("USER")
            .build();
        var admin = User.withDefaultPasswordEncoder()
            .username("admin@example.com")
            .password("password")
            .roles("ADMIN")
            .build();
        return new InMemoryUserDetailsManager(user, admin);
    }
}

// Reusable JWT token for tests
@TestComponent
public class TestJwtHelper {

    @Autowired
    private JwtService jwtService;

    public String tokenForUser(String email, String role) {
        var userDetails = User.builder()
            .username(email)
            .password("n/a")
            .roles(role)
            .build();
        return jwtService.generateToken(userDetails);
    }
}
```

---

## Testcontainers — Singleton Pattern (shared across tests)

```java
public abstract class AbstractIntegrationTest {

    static final PostgreSQLContainer<?> postgres;
    static final RedisContainer redis;

    static {
        postgres = new PostgreSQLContainer<>("postgres:16-alpine")
            .withReuse(true);
        redis = new RedisContainer(DockerImageName.parse("redis:7-alpine"))
            .withReuse(true);
        postgres.start();
        redis.start();
    }

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
        registry.add("spring.data.redis.host", redis::getHost);
        registry.add("spring.data.redis.port", redis::getFirstMappedPort);
    }
}

// Test class simply extends AbstractIntegrationTest
@SpringBootTest(webEnvironment = RANDOM_PORT)
class OrderIntegrationTest extends AbstractIntegrationTest {
    // no @Container needed — containers are shared
}
```

---

## Custom AssertJ Assertions

```java
public class UserAssert extends AbstractAssert<UserAssert, UserResponse> {

    public UserAssert(UserResponse actual) {
        super(actual, UserAssert.class);
    }

    public static UserAssert assertThat(UserResponse actual) {
        return new UserAssert(actual);
    }

    public UserAssert hasEmail(String email) {
        isNotNull();
        if (!actual.email().equals(email)) {
            failWithMessage("Expected email <%s> but was <%s>", email, actual.email());
        }
        return this;
    }

    public UserAssert hasRole(String role) {
        isNotNull();
        Assertions.assertThat(actual.role()).isEqualTo(role);
        return this;
    }
}

// Usage:
// assertThat(userResponse).hasEmail("john@example.com").hasRole("USER");
```

---

## @TestPropertySource — Override Properties

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.security.jwt.expiration=1000",   // very short for expiry test
    "app.feature.flag.enabled=true"
})
class JwtExpiryTest { ... }
```

---

## MockMvc with JWT Authentication

```java
// Helper method for tests requiring auth
private MockHttpServletRequestBuilder withJwt(MockHttpServletRequestBuilder request) {
    return request.header("Authorization", "Bearer " + testJwtHelper.tokenForUser("test@example.com", "USER"));
}

@Test
void protectedEndpoint_WithValidJwt_Returns200() throws Exception {
    given(userService.findById(1L)).willReturn(userResponse);

    mockMvc.perform(withJwt(get("/api/v1/users/1")))
        .andExpect(status().isOk());
}
```
