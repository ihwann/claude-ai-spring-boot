---
name: spring-boot-engineer
description: Generates Kotlin/Spring Boot 3.x configurations, creates REST controllers, implements Spring Security 6 with Kotlin DSL, sets up Spring Data JPA repositories, and configures coroutine-based async endpoints. Use when building Kotlin/Spring Boot 3.x applications, microservices, or async Kotlin applications; invoke for Spring Data JPA, Spring Security 6, WebFlux/Coroutines, Spring Cloud integration, Kotlin REST API design, or Microservices Kotlin architecture.
license: MIT
metadata:
  author: https://github.com/Jeffallan
  version: "2.0.0"
  domain: backend
  triggers: Spring Boot, Spring Framework, Spring Cloud, Spring Security, Spring Data JPA, Spring WebFlux, Coroutines, Kotlin REST API, Microservices Kotlin
  role: specialist
  scope: implementation
  output-format: code
  related-skills: kotlin-architect, database-optimizer, microservices-architect, devops-engineer
---

# Spring Boot Engineer (Kotlin)

## Core Workflow

1. **Analyze requirements** — Identify service boundaries, APIs, data models, security needs
2. **Design architecture** — Plan microservices, data access, cloud integration, security; confirm design before coding
3. **Implement** — Create services with constructor injection and layered architecture (see Quick Start below)
4. **Secure** — Add Spring Security Kotlin DSL, OAuth2, method security, CORS; verify security rules compile and pass tests
5. **Test** — Write unit (MockK), integration (TestContainers), and slice tests; run `./mvnw test` and confirm all pass
6. **Deploy** — Configure health checks and observability via Actuator; validate `/actuator/health` returns `UP`

## Reference Guide

Load detailed guidance based on context:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Web Layer | `references/web.md` | Controllers, REST APIs, validation, exception handling |
| Data Access | `references/data.md` | Spring Data JPA, repositories, transactions, projections |
| Security | `references/security.md` | Spring Security 6, OAuth2, JWT, method security |
| Cloud Native | `references/cloud.md` | Spring Cloud, Config, Discovery, Gateway, resilience |
| Testing | `references/testing.md` | MockK, @SpringBootTest, MockMvc, Testcontainers |

## Quick Start — Minimal Working Structure

A standard Kotlin/Spring Boot feature consists of these layers.

### Entity

```kotlin
@Entity
@Table(name = "products")
open class Product(  // open: required for Hibernate proxies (all-open plugin handles @Entity automatically)
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @field:NotBlank
    var name: String = "",

    @field:DecimalMin("0.0")
    var price: BigDecimal = BigDecimal.ZERO
)
// ⚠️ Do NOT use data class for JPA entities — copy() / equals() / hashCode() break Hibernate proxies
```

### Repository

```kotlin
interface ProductRepository : JpaRepository<Product, Long> {
    fun findByNameContainingIgnoreCase(name: String): List<Product>
}
```

### Service (constructor injection — primary constructor)

```kotlin
@Service
class ProductService(private val repo: ProductRepository) {  // no @Autowired needed

    @Transactional(readOnly = true)
    fun search(name: String): List<Product> =
        repo.findByNameContainingIgnoreCase(name)

    @Transactional
    fun create(request: ProductRequest): Product =
        repo.save(Product(name = request.name, price = request.price))

    @Transactional
    fun delete(id: Long) {
        if (!repo.existsById(id)) throw EntityNotFoundException("Product $id not found")
        repo.deleteById(id)
    }
}
```

### REST Controller

```kotlin
@RestController
@RequestMapping("/api/v1/products")
@Validated
class ProductController(private val service: ProductService) {

    @GetMapping
    fun search(@RequestParam(defaultValue = "") name: String): List<Product> =
        service.search(name)

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    fun create(@Valid @RequestBody request: ProductRequest): Product =
        service.create(request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    fun delete(@PathVariable id: Long) = service.delete(id)
}
```

### DTO (data class with @field: validation targets)

```kotlin
data class ProductRequest(
    @field:NotBlank(message = "Name is required")
    val name: String,

    @field:DecimalMin("0.0", message = "Price must be non-negative")
    val price: BigDecimal
)
// @field: is REQUIRED — without it, validation annotations on constructor params are ignored
```

### Global Exception Handler

```kotlin
@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(ex: MethodArgumentNotValidException): Map<String, String?> =
        ex.bindingResult.fieldErrors.associate { it.field to it.defaultMessage }

    @ExceptionHandler(EntityNotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(ex: EntityNotFoundException): Map<String, String?> =
        mapOf("error" to ex.message)
}
```

### Spring Security (Kotlin DSL)

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
            authorize(anyRequest, authenticated)
        }
        oauth2ResourceServer { jwt { } }
    }.build()
}
```

### Test Slice (MockK + SpringMockK)

```kotlin
@WebMvcTest(ProductController::class)
class ProductControllerTest {

    @Autowired lateinit var mockMvc: MockMvc
    @MockkBean lateinit var service: ProductService  // SpringMockK — not @MockBean

    @Test
    fun `createProduct - valid request - returns 201`() {
        val product = Product(name = "Widget", price = BigDecimal.TEN)
        every { service.create(any()) } returns product

        mockMvc.perform(
            post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name":"Widget","price":10.0}""")
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.name").value("Widget"))

        verify { service.create(any()) }
    }

    @Test
    fun `createProduct - blank name - returns 400`() {
        mockMvc.perform(
            post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""{"name":"","price":10.0}""")
        )
            .andExpect(status().isBadRequest)
    }
}
```

### Application Entry Point

```kotlin
@SpringBootApplication
@EnableJpaAuditing
class MyApplication

fun main(args: Array<String>) {
    runApplication<MyApplication>(*args)
}
```

## Constraints

### MUST DO

| Rule | Correct Pattern |
|------|----------------|
| Constructor injection | `class MyService(private val dep: Dep)` — primary constructor |
| Validate API input | `@Valid @RequestBody MyRequest` on every mutating endpoint |
| `@field:` validation | `data class Req(@field:NotBlank val name: String)` — without `@field:` validation is ignored |
| open class for JPA | `open class MyEntity(...)` or rely on all-open plugin for `@Entity` |
| Type-safe config | `@ConfigurationProperties(prefix = "app")` bound to a data class |
| Appropriate stereotype | `@Service` for business logic, `@Repository` for data, `@RestController` for HTTP |
| Transaction scope | `@Transactional` on multi-step writes; `@Transactional(readOnly = true)` on reads |
| MockK for tests | `@MockkBean` in Spring tests, `mockk<Type>()` or `@MockK` in unit tests |
| Hide internals | Catch domain exceptions in `@RestControllerAdvice`; return problem details |
| Externalize secrets | Use environment variables or Spring Cloud Config — never `application.properties` |

### MUST NOT DO
- Use `!!` operator (use `?.`, `?:`, `requireNotNull()` instead)
- Use `data class` for JPA entities (Hibernate proxy / equals / hashCode issues)
- Omit `@field:` on validation annotations in data class constructor params
- Use field injection (`@Autowired` on fields)
- Skip input validation on API endpoints
- Use deprecated Spring Boot 2.x patterns (e.g., `WebSecurityConfigurerAdapter`)
- Call blocking APIs from coroutine context without `Dispatchers.IO`
- Use `@MockBean` with Mockito in Kotlin tests — use `@MockkBean` (SpringMockK)
