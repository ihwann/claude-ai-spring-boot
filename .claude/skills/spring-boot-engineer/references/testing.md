# Testing - Kotlin/Spring Boot (MockK)

> Use **MockK** (not Mockito) for all mocking in Kotlin projects.
> Use **`@MockkBean`** (SpringMockK) in Spring context tests — not `@MockBean`.

## Unit Testing with JUnit 5 + MockK

```kotlin
import io.mockk.*
import io.mockk.impl.annotations.*
import io.mockk.junit5.MockKExtension
import org.assertj.core.api.Assertions.*
import org.junit.jupiter.api.*
import org.junit.jupiter.api.extension.ExtendWith

@ExtendWith(MockKExtension::class)
@DisplayName("User Service Tests")
class UserServiceTest {

    @MockK
    lateinit var userRepository: UserRepository

    @MockK
    lateinit var passwordEncoder: PasswordEncoder

    @InjectMockKs
    lateinit var userService: UserService

    @Test
    @DisplayName("Should create user successfully")
    fun `create - valid request - saves user and returns response`() {
        // Given
        val request = CreateUserRequest(email = "test@example.com", password = "Password123", username = "testuser", age = 25)
        val savedUser = User(id = 1L, email = request.email, username = request.username)

        every { userRepository.existsByEmail(request.email) } returns false
        every { passwordEncoder.encode(request.password) } returns "encodedPassword"
        every { userRepository.save(any()) } returns savedUser

        // When
        val response = userService.create(request)

        // Then
        assertThat(response).isNotNull
        assertThat(response.email).isEqualTo(request.email)
        verify { userRepository.existsByEmail(request.email) }
        verify { passwordEncoder.encode(request.password) }
        verify { userRepository.save(any()) }
        confirmVerified(userRepository, passwordEncoder)
    }

    @Test
    @DisplayName("Should throw when email already exists")
    fun `create - duplicate email - throws without saving`() {
        // Given
        val request = CreateUserRequest(email = "test@example.com", password = "Password123", username = "testuser", age = 25)
        every { userRepository.existsByEmail(request.email) } returns true

        // When / Then
        assertThatThrownBy { userService.create(request) }
            .isInstanceOf(DuplicateResourceException::class.java)
            .hasMessageContaining("Email already registered")

        verify(exactly = 0) { userRepository.save(any()) }  // assert save was NOT called
    }
}
```

## Integration Testing with @SpringBootTest

```kotlin
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@ActiveProfiles("test")
@TestMethodOrder(MethodOrderer.OrderAnnotation::class)
class UserIntegrationTest {

    @Autowired
    private lateinit var restTemplate: TestRestTemplate

    @Autowired
    private lateinit var userRepository: UserRepository

    @BeforeEach
    fun setUp() {
        userRepository.deleteAll()
    }

    @Test
    @Order(1)
    @DisplayName("Should create user via API")
    fun `createUser - valid request - returns 201`() {
        val request = CreateUserRequest(email = "test@example.com", password = "Password123", username = "testuser", age = 25)

        val response = restTemplate.postForEntity("/api/v1/users", request, UserResponse::class.java)

        assertThat(response.statusCode).isEqualTo(HttpStatus.CREATED)
        assertThat(response.body).isNotNull
        assertThat(response.body!!.email).isEqualTo(request.email)
        assertThat(response.headers.location).isNotNull
    }

    @Test
    @Order(2)
    @DisplayName("Should return validation error for invalid request")
    fun `createUser - invalid input - returns 400`() {
        val request = CreateUserRequest(email = "invalid-email", password = "short", username = "u", age = 15)

        val response = restTemplate.postForEntity("/api/v1/users", request, ValidationErrorResponse::class.java)

        assertThat(response.statusCode).isEqualTo(HttpStatus.BAD_REQUEST)
        assertThat(response.body!!.errors).isNotEmpty
    }
}
```

## Web Layer Testing (SpringMockK)

```kotlin
@WebMvcTest(UserController::class)
@Import(SecurityConfig::class)
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @MockkBean  // SpringMockK — use this instead of @MockBean
    private lateinit var userService: UserService

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @Test
    @WithMockUser(roles = ["ADMIN"])
    @DisplayName("Should get all users")
    fun `getAll - admin user - returns 200 with list`() {
        val users = listOf(
            UserResponse(id = 1L, email = "user1@example.com", username = "user1"),
            UserResponse(id = 2L, email = "user2@example.com", username = "user2")
        )
        every { userService.findAll(any()) } returns PageImpl(users)

        mockMvc.perform(get("/api/v1/users").contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.content").isArray)
            .andExpect(jsonPath("$.content.length()").value(2))
            .andExpect(jsonPath("$.content[0].email").value("user1@example.com"))
    }

    @Test
    @WithMockUser(roles = ["ADMIN"])
    @DisplayName("Should create user")
    fun `create - valid request - returns 201`() {
        val request = CreateUserRequest(email = "test@example.com", password = "Password123", username = "testuser", age = 25)
        val response = UserResponse(id = 1L, email = request.email, username = request.username)
        every { userService.create(any()) } returns response

        mockMvc.perform(
            post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isCreated)
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.email").value(request.email))
    }

    @Test
    @WithMockUser(roles = ["USER"])
    @DisplayName("Should return 403 for non-admin user")
    fun `getAll - user role - returns 403`() {
        mockMvc.perform(get("/api/v1/users").contentType(MediaType.APPLICATION_JSON))
            .andExpect(status().isForbidden)
    }
}
```

## Data JPA Testing

```kotlin
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
@ActiveProfiles("test")
class UserRepositoryTest {

    @Autowired
    private lateinit var userRepository: UserRepository

    @Autowired
    private lateinit var entityManager: TestEntityManager

    @Test
    @DisplayName("Should find user by email")
    fun `findByEmail - existing user - returns user`() {
        val user = User(email = "test@example.com", password = "password", username = "testuser", active = true)
        entityManager.persistAndFlush(user)

        val found = userRepository.findByEmail("test@example.com")

        assertThat(found).isPresent
        assertThat(found.get().email).isEqualTo("test@example.com")
    }

    @Test
    @DisplayName("Should check if email exists")
    fun `existsByEmail - existing email - returns true`() {
        val user = User(email = "test@example.com", password = "password", username = "testuser", active = true)
        entityManager.persistAndFlush(user)

        assertThat(userRepository.existsByEmail("test@example.com")).isTrue
    }
}
```

## Testcontainers

```kotlin
@SpringBootTest
@Testcontainers
@ActiveProfiles("test")
class UserServiceIntegrationTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:17-alpine").apply {
            withDatabaseName("testdb")
            withUsername("test")
            withPassword("test")
        }

        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Autowired
    private lateinit var userService: UserService

    @Autowired
    private lateinit var userRepository: UserRepository

    @BeforeEach
    fun setUp() {
        userRepository.deleteAll()
    }

    @Test
    fun `createAndFind - full lifecycle - succeeds`() {
        val request = CreateUserRequest(email = "test@example.com", password = "Password123", username = "testuser", age = 25)

        val created = userService.create(request)
        val found = userService.findById(created.id)

        assertThat(found).isNotNull
        assertThat(found.email).isEqualTo(request.email)
    }
}
```

## Testing Reactive/Coroutine Endpoints

```kotlin
@WebFluxTest(UserReactiveController::class)
class UserReactiveControllerTest {

    @Autowired
    private lateinit var webTestClient: WebTestClient

    @MockkBean
    private lateinit var userService: UserReactiveService

    @Test
    @DisplayName("Should get user reactively")
    fun `getById - existing user - returns 200`() {
        val user = UserResponse(id = 1L, email = "test@example.com", username = "testuser")
        coEvery { userService.findById(1L) } returns user

        webTestClient.get()
            .uri("/api/v1/users/{id}", 1L)
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk
            .expectBody(UserResponse::class.java)
            .value { response ->
                assertThat(response.id).isEqualTo(1L)
                assertThat(response.email).isEqualTo("test@example.com")
            }
    }
}
```

## Test Fixtures (Kotlin Object)

```kotlin
object UserTestFixtures {
    fun aUser(
        id: Long = 1L,
        email: String = "test@example.com",
        username: String = "testuser",
        active: Boolean = true
    ) = User(id = id, email = email, username = username, active = active)

    fun aCreateRequest(
        email: String = "test@example.com",
        password: String = "Password123",
        username: String = "testuser",
        age: Int = 25
    ) = CreateUserRequest(email = email, password = password, username = username, age = age)
}

// Usage
val user = UserTestFixtures.aUser(email = "custom@example.com")
val inactive = UserTestFixtures.aUser(active = false)
```

## Quick Reference

| Annotation / Function | Purpose |
|----------------------|---------|
| `@ExtendWith(MockKExtension::class)` | Enable MockK for JUnit 5 |
| `@MockK` | Create MockK mock |
| `@InjectMockKs` | Inject mocks into subject |
| `@MockkBean` | Mock Spring bean (SpringMockK, use in `@WebMvcTest`) |
| `every { } returns` | Stub method call |
| `coEvery { } returns` | Stub suspend function |
| `verify { }` | Verify method was called |
| `coVerify { }` | Verify suspend method was called |
| `verify(exactly = 0) { }` | Assert method was NOT called |
| `confirmVerified()` | No unexpected calls |
| `every { } throws` | Stub exception throw |
| `@SpringBootTest` | Full application context |
| `@WebMvcTest` | Test MVC layer only |
| `@DataJpaTest` | Test JPA repositories |
| `@WithMockUser` | Mock authenticated user |

## Testing Best Practices

- Write tests following AAA pattern (Arrange, Act, Assert)
- Use Kotlin backtick test names: `` fun `method - scenario - expected result`() ``
- Use `confirmVerified()` to ensure no unexpected mock interactions
- Mock external dependencies with MockK, use real DB with Testcontainers
- Achieve 85%+ code coverage
- Test happy path AND edge cases AND error scenarios
- Use `@Transactional` for test data cleanup in integration tests
- Separate unit tests from integration tests
- Use `runTest { }` for coroutine testing
