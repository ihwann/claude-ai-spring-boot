# Async Programming: Coroutines + WebFlux (Kotlin)

> **Kotlin-first approach**: Prefer `suspend fun` + `Flow` over raw `Mono`/`Flux`.
> Use coroutines for readability; bridge to Reactor only when required by Spring APIs.

## Coroutine-based Controller (Preferred)

```kotlin
package pl.piomin.services.presentation.rest

import kotlinx.coroutines.flow.Flow
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/users")
class UserController(private val userService: UserService) {

    // suspend fun — non-blocking, coroutine-based
    @GetMapping("/{id}")
    suspend fun getUserById(@PathVariable id: Long): ResponseEntity<UserResponse> =
        userService.findById(id)
            ?.let { ResponseEntity.ok(it) }
            ?: ResponseEntity.notFound().build()

    // Flow — streaming response
    @GetMapping
    fun getAllUsers(): Flow<UserResponse> = userService.findAll()

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    suspend fun createUser(@RequestBody @Valid request: UserRequest): UserResponse =
        userService.create(request)

    @PutMapping("/{id}")
    suspend fun updateUser(
        @PathVariable id: Long,
        @RequestBody @Valid request: UserRequest
    ): UserResponse = userService.update(id, request)

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    suspend fun deleteUser(@PathVariable id: Long) = userService.delete(id)
}
```

## Coroutine Service Layer

```kotlin
package pl.piomin.services.application.service

import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.map
import kotlinx.coroutines.withContext
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional

@Service
class UserService(private val userRepository: UserRepository) {

    // Use Dispatchers.IO for JPA/JDBC blocking calls inside suspend functions
    @Transactional(readOnly = true)
    suspend fun findById(id: Long): UserResponse? = withContext(Dispatchers.IO) {
        userRepository.findById(id).map { UserResponse.from(it) }.orElse(null)
    }

    fun findAll(): Flow<UserResponse> =
        userRepository.findAllAsFlow().map { UserResponse.from(it) }

    @Transactional
    suspend fun create(request: UserRequest): UserResponse = withContext(Dispatchers.IO) {
        val saved = userRepository.save(request.toEntity())
        UserResponse.from(saved)
    }

    @Transactional
    suspend fun delete(id: Long): Unit = withContext(Dispatchers.IO) {
        if (!userRepository.existsById(id))
            throw EntityNotFoundException("User $id not found")
        userRepository.deleteById(id)
    }
}
```

## R2DBC Repository (Fully Reactive)

```kotlin
package pl.piomin.services.domain.repository

import kotlinx.coroutines.flow.Flow
import org.springframework.data.r2dbc.repository.Query
import org.springframework.data.repository.kotlin.CoroutineCrudRepository

// CoroutineCrudRepository: Spring Data R2DBC with Kotlin coroutines
interface UserRepository : CoroutineCrudRepository<User, Long> {

    suspend fun findByEmail(email: String): User?

    fun findByActiveTrue(): Flow<User>

    @Query("""
        SELECT u.* FROM users u
        WHERE u.email LIKE CONCAT('%', :domain, '%')
        ORDER BY u.created_at DESC
    """)
    fun findByEmailDomain(domain: String): Flow<User>
}
```

## R2DBC Entity

```kotlin
package pl.piomin.services.domain.model

import org.springframework.data.annotation.*
import org.springframework.data.relational.core.mapping.Table
import java.time.Instant

@Table("users")
data class User(  // R2DBC entities CAN be data class (no Hibernate proxies)
    @Id val id: Long? = null,
    val email: String,
    val username: String,
    val active: Boolean = true,
    @CreatedDate val createdAt: Instant? = null,
    @LastModifiedDate val updatedAt: Instant? = null
)
```

> ⚠️ **JPA vs R2DBC entities**: `data class` is fine for R2DBC. For JPA use `open class`.

## R2DBC Configuration

```yaml
spring:
  r2dbc:
    url: r2dbc:postgresql://localhost:5432/demo
    username: ${DB_USER}
    password: ${DB_PASSWORD}
    pool:
      initial-size: 10
      max-size: 20
      max-idle-time: 30m
  data:
    r2dbc:
      repositories:
        enabled: true
```

## WebClient for External APIs (Kotlin)

```kotlin
package pl.piomin.services.infrastructure.client

import org.springframework.stereotype.Component
import org.springframework.web.reactive.function.client.WebClient
import org.springframework.web.reactive.function.client.awaitBody  // Kotlin coroutine extension
import java.time.Duration

@Component
class ExternalUserClient(private val webClient: WebClient) {

    // suspend fun using awaitBody (coroutine extension on WebClient)
    suspend fun getUser(id: Long): ExternalUserDto =
        webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .awaitBody()  // suspend extension — no .block() needed

    suspend fun createUser(user: ExternalUserDto): ExternalUserDto =
        webClient.post()
            .uri("/users")
            .bodyValue(user)
            .retrieve()
            .awaitBody()
}

@Configuration
class WebClientConfig {
    @Bean
    fun webClient(builder: WebClient.Builder): WebClient =
        builder
            .baseUrl("https://api.example.com")
            .defaultHeader("User-Agent", "My Service")
            .build()
}
```

## Kotlin Flow Operations

```kotlin
import kotlinx.coroutines.flow.*

// Transform
val names: Flow<String> = userFlow
    .filter { it.active }
    .map { it.name }
    .take(10)

// Collect
userFlow.collect { user ->
    println(user.email)
}

// Combine flows
val combined: Flow<Pair<User, Order>> = userFlow.zip(orderFlow) { user, order ->
    user to order
}

// Error handling
val safe: Flow<User> = userFlow
    .catch { e -> log.error("Error in flow", e) }
    .onCompletion { log.info("Flow completed") }

// Convert from Reactor Flux
val kotlinFlow: Flow<User> = reactorFlux.asFlow()

// Convert to Reactor Flux
val reactorFlux: Flux<User> = kotlinFlow.asFlux()
```

## Bridging Coroutines ↔ Reactor

```kotlin
import kotlinx.coroutines.reactor.mono
import kotlinx.coroutines.reactor.flux
import reactor.core.publisher.Mono
import reactor.core.publisher.Flux

// Coroutine → Mono
fun userMono(id: Long): Mono<User> = mono {
    userRepository.findById(id) ?: throw EntityNotFoundException("User $id not found")
}

// Coroutine → Flux
fun userFlux(): Flux<User> = flux {
    userRepository.findAll().collect { send(it) }
}

// Mono → suspend
suspend fun fromMono(): User = userMono(1L).awaitSingle()

// Flux → Flow
val flow: Flow<User> = userFlux().asFlow()
```

## Testing Coroutines (runTest)

```kotlin
import io.mockk.coEvery
import io.mockk.coVerify
import kotlinx.coroutines.test.runTest
import org.junit.jupiter.api.Test

class UserServiceTest {

    @MockK lateinit var repository: UserRepository
    @InjectMockKs lateinit var service: UserService

    @Test
    fun `findById - suspend function - returns user`() = runTest {
        coEvery { repository.findById(1L) } returns Optional.of(User(id = 1L))

        val result = service.findById(1L)

        assertThat(result).isNotNull
        coVerify { repository.findById(1L) }
    }

    @Test
    fun `findAll - flow - emits users`() = runTest {
        val users = listOf(User(id = 1L), User(id = 2L))
        every { repository.findAllAsFlow() } returns users.asFlow()

        val result = service.findAll().toList()

        assertThat(result).hasSize(2)
    }
}
```

## Quick Reference

| Pattern | Code |
|---------|------|
| Suspend controller | `suspend fun get(@PathVariable id: Long): ResponseEntity<T>` |
| Flow endpoint | `fun getAll(): Flow<T>` |
| Blocking in suspend | `withContext(Dispatchers.IO) { blockingCall() }` |
| R2DBC repository | `CoroutineCrudRepository<T, ID>` |
| WebClient coroutine | `.retrieve().awaitBody<T>()` |
| Mono → suspend | `mono.awaitSingle()` |
| Flux → Flow | `flux.asFlow()` |
| Test suspend | `@Test fun \`name\`() = runTest { ... }` |
| Mock suspend | `coEvery { } returns` / `coVerify { }` |
