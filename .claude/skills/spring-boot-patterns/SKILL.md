---
name: spring-boot-patterns
description: Kotlin/Spring Boot best practices and patterns. Use when creating controllers, services, repositories, or when user asks about Spring Boot architecture, REST APIs, exception handling, or JPA patterns in Kotlin.
---

# Spring Boot Patterns Skill (Kotlin)

Best practices and patterns for Kotlin/Spring Boot applications.

## When to Use
- User says "create controller" / "add service" / "Spring Boot help"
- Reviewing Kotlin Spring Boot code
- Setting up new Kotlin/Spring Boot project structure

## Project Structure

```
src/main/kotlin/pl/piomin/services/
├── Application.kt                  # @SpringBootApplication
├── config/                         # Configuration classes
│   ├── SecurityConfig.kt           # Kotlin DSL security
│   └── WebConfig.kt
├── controller/                     # REST controllers
│   └── UserController.kt
├── service/                        # Business logic
│   └── UserService.kt
├── repository/                     # Data access
│   └── UserRepository.kt
├── model/                          # JPA Entities (open class)
│   └── User.kt
├── dto/                            # Data transfer objects (data class)
│   ├── request/
│   │   └── CreateUserRequest.kt
│   └── response/
│       └── UserResponse.kt
└── exception/                      # Custom exceptions + handler
    ├── ResourceNotFoundException.kt
    └── GlobalExceptionHandler.kt
```

---

## Controller Patterns

### REST Controller Template

```kotlin
@RestController
@RequestMapping("/api/v1/users")
class UserController(private val userService: UserService) {  // constructor injection

    @GetMapping
    fun getAll(): ResponseEntity<List<UserResponse>> =
        ResponseEntity.ok(userService.findAll())

    @GetMapping("/{id}")
    fun getById(@PathVariable id: Long): ResponseEntity<UserResponse> =
        ResponseEntity.ok(userService.findById(id))

    @PostMapping
    fun create(@Valid @RequestBody request: CreateUserRequest): ResponseEntity<UserResponse> {
        val created = userService.create(request)
        val location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(created.id)
            .toUri()
        return ResponseEntity.created(location).body(created)
    }

    @PutMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @Valid @RequestBody request: UpdateUserRequest
    ): ResponseEntity<UserResponse> =
        ResponseEntity.ok(userService.update(id, request))

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) = userService.delete(id)
}
```

### Controller Best Practices

| Practice | Example |
|----------|---------|
| Versioned API | `/api/v1/users` |
| Plural nouns | `/users` not `/user` |
| HTTP methods | GET=read, POST=create, PUT=update, DELETE=delete |
| Status codes | 200=OK, 201=Created, 204=NoContent, 404=NotFound |
| Validation | `@Valid` on request body |

### ❌ Anti-patterns

```kotlin
// ❌ Business logic in controller
@PostMapping
fun create(@RequestBody user: User): User {
    user.createdAt = LocalDateTime.now()  // logic belongs in service
    return userRepository.save(user)      // direct repo access
}

// ❌ Returning entity directly (exposes internals + Hibernate proxy issues)
@GetMapping("/{id}")
fun getById(@PathVariable id: Long): User =
    userRepository.findById(id).get()
```

---

## Service Patterns

### Service with Constructor Injection

```kotlin
@Service
@Transactional(readOnly = true)  // default read-only
class UserService(private val userRepository: UserRepository) {

    fun findAll(): List<UserResponse> =
        userRepository.findAll().map { UserResponse.from(it) }

    fun findById(id: Long): UserResponse =
        userRepository.findById(id)
            .map { UserResponse.from(it) }
            .orElseThrow { ResourceNotFoundException("User", id) }

    @Transactional
    fun create(request: CreateUserRequest): UserResponse =
        userRepository.save(request.toEntity()).let { UserResponse.from(it) }

    @Transactional
    fun update(id: Long, request: UpdateUserRequest): UserResponse {
        val user = userRepository.findById(id)
            .orElseThrow { ResourceNotFoundException("User", id) }
        user.name = request.name
        user.email = request.email
        return UserResponse.from(user)
    }

    @Transactional
    fun delete(id: Long) {
        if (!userRepository.existsById(id))
            throw ResourceNotFoundException("User", id)
        userRepository.deleteById(id)
    }
}
```

### Service Best Practices

- Constructor injection (primary constructor) — no `@Autowired` needed
- `@Transactional(readOnly = true)` at class level
- `@Transactional` for write methods (overrides class-level)
- Throw domain exceptions, not generic ones
- Use companion object or extension functions for entity ↔ DTO conversion

---

## Repository Patterns

### JPA Repository

```kotlin
interface UserRepository : JpaRepository<User, Long> {

    // Derived query
    fun findByEmail(email: String): Optional<User>

    fun findByActiveTrue(): List<User>

    // Custom query
    @Query("SELECT u FROM User u WHERE u.department.id = :deptId")
    fun findByDepartmentId(@Param("deptId") departmentId: Long): List<User>

    // Projection
    @EntityGraph(attributePaths = ["roles"])
    fun findByEmailWithRoles(email: String): Optional<User>

    // Existence check (more efficient than findBy)
    fun existsByEmail(email: String): Boolean

    fun countByActiveTrue(): Long
}
```

---

## DTO Patterns

### Request DTO (data class with @field: validation)

```kotlin
data class CreateUserRequest(
    @field:NotBlank(message = "Name is required")
    @field:Size(min = 2, max = 100)
    val name: String,

    @field:NotBlank
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:NotNull
    @field:Min(18)
    val age: Int
) {
    fun toEntity(): User = User(name = name, email = email, age = age)
}
```

### Response DTO (data class with companion factory)

```kotlin
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val createdAt: LocalDateTime?
) {
    companion object {
        fun from(user: User) = UserResponse(
            id = user.id,
            name = user.name,
            email = user.email,
            createdAt = user.createdAt
        )
    }
}
```

---

## Exception Handling

### Custom Exceptions

```kotlin
class ResourceNotFoundException(resource: String, id: Long) :
    RuntimeException("$resource not found with id: $id")

class BusinessException(val code: String, message: String) :
    RuntimeException(message)

class DuplicateResourceException(resource: String, field: String) :
    RuntimeException("$resource already exists with $field")
```

### Global Exception Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    private val log = LoggerFactory.getLogger(javaClass)

    @ExceptionHandler(ResourceNotFoundException::class)
    fun handleNotFound(ex: ResourceNotFoundException): ResponseEntity<ErrorResponse> {
        log.warn("Resource not found: {}", ex.message)
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(ErrorResponse("NOT_FOUND", ex.message ?: "Not found"))
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ResponseEntity<ErrorResponse> {
        val errors = ex.bindingResult.fieldErrors
            .joinToString(", ") { "${it.field}: ${it.defaultMessage}" }
        return ResponseEntity.badRequest()
            .body(ErrorResponse("VALIDATION_ERROR", errors))
    }

    @ExceptionHandler(Exception::class)
    fun handleGeneric(ex: Exception): ResponseEntity<ErrorResponse> {
        log.error("Unexpected error", ex)
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(ErrorResponse("INTERNAL_ERROR", "An unexpected error occurred"))
    }
}

data class ErrorResponse(val code: String, val message: String)
```

---

## Configuration Patterns

### Application Properties

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false  # Always disable! Prevents lazy loading outside transaction
    show-sql: false

app:
  jwt:
    secret: ${JWT_SECRET}
    expiration: 86400000
```

### Configuration Properties (data class)

```kotlin
@Configuration
@ConfigurationProperties(prefix = "app.jwt")
@Validated
data class JwtProperties(
    @field:NotBlank val secret: String = "",
    @field:Min(60000) val expiration: Long = 0
)
```

---

## Spring Security (Kotlin DSL)

```kotlin
@Configuration
@EnableMethodSecurity
class SecurityConfig {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain = http {
        csrf { disable() }
        sessionManagement { sessionCreationPolicy = SessionCreationPolicy.STATELESS }
        authorizeHttpRequests {
            authorize("/actuator/health", permitAll)
            authorize("/api/public/**", permitAll)
            authorize("/api/admin/**", hasRole("ADMIN"))
            authorize(anyRequest, authenticated)
        }
        oauth2ResourceServer { jwt { } }
    }.build()
}
```

---

## Testing Patterns

### Controller Test (MockK + SpringMockK)

```kotlin
@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired lateinit var mockMvc: MockMvc
    @MockkBean lateinit var userService: UserService  // SpringMockK

    @Test
    fun `getById - existing user - returns 200`() {
        val response = UserResponse(1L, "John", "john@example.com", null)
        every { userService.findById(1L) } returns response

        mockMvc.perform(get("/api/v1/users/1"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.name").value("John"))
    }

    @Test
    fun `getById - not found - returns 404`() {
        every { userService.findById(999L) } throws ResourceNotFoundException("User", 999L)

        mockMvc.perform(get("/api/v1/users/999"))
            .andExpect(status().isNotFound)
    }
}
```

### Service Test (MockK)

```kotlin
@ExtendWith(MockKExtension::class)
class UserServiceTest {

    @MockK lateinit var userRepository: UserRepository
    @InjectMockKs lateinit var userService: UserService

    @Test
    fun `findById - existing user - returns response`() {
        val user = User(id = 1L, name = "John", email = "john@example.com")
        every { userRepository.findById(1L) } returns Optional.of(user)

        val result = userService.findById(1L)

        assertThat(result.name).isEqualTo("John")
        verify { userRepository.findById(1L) }
        confirmVerified(userRepository)
    }

    @Test
    fun `findById - not found - throws exception`() {
        every { userRepository.findById(any()) } returns Optional.empty()

        assertThatThrownBy { userService.findById(1L) }
            .isInstanceOf(ResourceNotFoundException::class.java)
    }
}
```

### Integration Test (TestContainers)

```kotlin
@SpringBootTest
@AutoConfigureMockMvc
@Testcontainers
class UserIntegrationTest {

    @Container
    companion object {
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:17-alpine")
            .apply { start() }

        @JvmStatic
        @DynamicPropertySource
        fun configureProperties(registry: DynamicPropertyRegistry) {
            registry.add("spring.datasource.url", postgres::getJdbcUrl)
            registry.add("spring.datasource.username", postgres::getUsername)
            registry.add("spring.datasource.password", postgres::getPassword)
        }
    }

    @Autowired lateinit var mockMvc: MockMvc

    @Test
    fun `createUser - valid request - returns 201`() {
        mockMvc.perform(
            post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name": "John", "email": "john@example.com", "age": 25}""")
        )
            .andExpect(status().isCreated)
    }
}
```

---

## Quick Reference Card

| Layer | Responsibility | Kotlin Type |
|-------|---------------|-------------|
| Controller | HTTP handling, validation | `class` with `@RestController` |
| Service | Business logic, transactions | `class` with `@Service` |
| Repository | Data access | `interface` extends `JpaRepository` |
| Entity | JPA model | `open class` with `@Entity` |
| DTO | Data transfer | `data class` with `@field:` validation |
| Config | Configuration | `@Configuration` + Kotlin DSL |
| Exception | Error handling | `@RestControllerAdvice` |

| Annotation | Note |
|------------|------|
| `@MockkBean` | Use instead of `@MockBean` in Spring tests |
| `@field:NotBlank` | Always use `@field:` target on data class params |
| `open class` | JPA entities must be open (all-open plugin handles automatically) |
| `http { }` | Kotlin DSL for SecurityFilterChain |
