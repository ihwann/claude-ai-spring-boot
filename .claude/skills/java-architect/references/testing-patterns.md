# Testing Patterns (Kotlin + MockK)

## Unit Testing with JUnit 5 + MockK

```kotlin
package pl.piomin.services.application.service

import io.mockk.*
import io.mockk.impl.annotations.InjectMockKs
import io.mockk.impl.annotations.MockK
import io.mockk.junit5.MockKExtension
import org.assertj.core.api.Assertions.*
import org.junit.jupiter.api.*
import org.junit.jupiter.api.extension.ExtendWith
import org.junit.jupiter.params.ParameterizedTest
import org.junit.jupiter.params.provider.ValueSource
import java.util.Optional

@ExtendWith(MockKExtension::class)
@DisplayName("User Service Tests")
class UserServiceTest {

    @MockK
    lateinit var userRepository: UserRepository

    @InjectMockKs
    lateinit var userService: UserService

    private val testUser = User(id = 1L, email = "test@example.com", name = "Test User")
    private val userRequest = CreateUserRequest(email = "test@example.com", name = "Test User")
    private val userResponse = UserResponse(id = 1L, email = "test@example.com", name = "Test User")

    @Test
    @DisplayName("Should find user by ID successfully")
    fun `findById - existing user - returns response`() {
        // Given
        every { userRepository.findById(1L) } returns Optional.of(testUser)

        // When
        val result = userService.findById(1L)

        // Then
        assertThat(result.email).isEqualTo("test@example.com")
        verify { userRepository.findById(1L) }
        confirmVerified(userRepository)  // ensures no unexpected interactions
    }

    @Test
    @DisplayName("Should throw exception when user not found")
    fun `findById - not found - throws EntityNotFoundException`() {
        // Given
        every { userRepository.findById(any()) } returns Optional.empty()

        // When / Then
        assertThatThrownBy { userService.findById(999L) }
            .isInstanceOf(EntityNotFoundException::class.java)
            .hasMessageContaining("User not found")

        verify { userRepository.findById(999L) }
    }

    @Test
    @DisplayName("Should create user successfully")
    fun `create - valid request - saves and returns response`() {
        // Given
        every { userRepository.save(any()) } returns testUser

        // When
        val result = userService.create(userRequest)

        // Then
        assertThat(result.id).isEqualTo(1L)
        verify { userRepository.save(any()) }
    }

    @Test
    @DisplayName("Should not call save when email already exists")
    fun `create - duplicate email - throws exception without saving`() {
        // Given
        every { userRepository.existsByEmail(userRequest.email) } returns true

        // When / Then
        assertThatThrownBy { userService.create(userRequest) }
            .isInstanceOf(DuplicateResourceException::class.java)

        verify(exactly = 0) { userRepository.save(any()) }  // verify not called
    }

    @ParameterizedTest
    @ValueSource(strings = ["admin", "user", "moderator"])
    @DisplayName("Should validate different user roles")
    fun `validateRole - valid roles - passes`(role: String) {
        assertThat(role).isNotBlank
    }
}
```

## Integration Testing with TestContainers

```kotlin
package pl.piomin.services.integration

import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.BeforeEach
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.boot.test.web.client.TestRestTemplate
import org.springframework.http.HttpStatus
import org.springframework.test.context.DynamicPropertyRegistry
import org.springframework.test.context.DynamicPropertySource
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@Testcontainers
class UserIntegrationTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:17-alpine")
            .apply {
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
    private lateinit var restTemplate: TestRestTemplate

    @Test
    fun `createAndRetrieveUser - full round trip - succeeds`() {
        // Create
        val request = CreateUserRequest(email = "test@example.com", name = "Test User")
        val createResponse = restTemplate.postForEntity(
            "/api/users",
            request,
            UserResponse::class.java
        )
        assertThat(createResponse.statusCode).isEqualTo(HttpStatus.CREATED)
        val userId = createResponse.body!!.id

        // Retrieve
        val getResponse = restTemplate.getForEntity(
            "/api/users/$userId",
            UserResponse::class.java
        )
        assertThat(getResponse.statusCode).isEqualTo(HttpStatus.OK)
        assertThat(getResponse.body!!.email).isEqualTo("test@example.com")
    }
}
```

## Repository Testing

```kotlin
package pl.piomin.services.domain.repository

import com.ninja_squad.springmockk.MockkBean
import org.assertj.core.api.Assertions.assertThat
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest
import org.springframework.boot.test.autoconfigure.orm.jpa.TestEntityManager
import org.testcontainers.containers.PostgreSQLContainer
import org.testcontainers.junit.jupiter.Container
import org.testcontainers.junit.jupiter.Testcontainers

@DataJpaTest
@Testcontainers
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class UserRepositoryTest {

    companion object {
        @Container
        @JvmStatic
        val postgres = PostgreSQLContainer<Nothing>("postgres:17-alpine")
    }

    @Autowired
    private lateinit var entityManager: TestEntityManager

    @Autowired
    private lateinit var userRepository: UserRepository

    @Test
    fun `findByEmail - existing user - returns user`() {
        // Given
        val user = User(email = "test@example.com", name = "Test User")
        entityManager.persistAndFlush(user)

        // When
        val found = userRepository.findByEmail("test@example.com")

        // Then
        assertThat(found).isPresent
        assertThat(found.get().email).isEqualTo("test@example.com")
    }

    @Test
    fun `countByActiveTrue - mixed users - counts only active`() {
        entityManager.persist(User(email = "active@example.com", active = true))
        entityManager.persist(User(email = "inactive@example.com", active = false))
        entityManager.flush()

        assertThat(userRepository.countByActiveTrue()).isEqualTo(1)
    }
}
```

## REST Controller Testing (SpringMockK)

```kotlin
package pl.piomin.services.presentation.rest

import com.fasterxml.jackson.databind.ObjectMapper
import com.ninja_squad.springmockk.MockkBean
import io.mockk.every
import io.mockk.verify
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest
import org.springframework.http.MediaType
import org.springframework.security.test.context.support.WithMockUser
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*
import org.springframework.test.web.servlet.result.MockMvcResultMatchers.*

@WebMvcTest(UserController::class)
class UserControllerTest {

    @Autowired
    private lateinit var mockMvc: MockMvc

    @Autowired
    private lateinit var objectMapper: ObjectMapper

    @MockkBean  // SpringMockK — NOT @MockBean (Mockito)
    private lateinit var userService: UserService

    @Test
    @WithMockUser
    fun `getById - existing user - returns 200`() {
        val response = UserResponse(id = 1L, email = "test@example.com", name = "Test")
        every { userService.findById(1L) } returns response

        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk)
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.email").value("test@example.com"))

        verify { userService.findById(1L) }
    }

    @Test
    @WithMockUser
    fun `create - valid request - returns 201`() {
        val request = CreateUserRequest(email = "test@example.com", name = "Test")
        val response = UserResponse(id = 1L, email = "test@example.com", name = "Test")
        every { userService.create(any()) } returns response

        mockMvc.perform(
            post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request))
        )
            .andExpect(status().isCreated)
            .andExpect(jsonPath("$.id").value(1))
    }

    @Test
    fun `getById - unauthenticated - returns 401`() {
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isUnauthorized)
    }
}
```

## Test Data Builders (Kotlin Object)

```kotlin
package pl.piomin.services.test

object UserTestFixtures {
    fun aUser(
        id: Long = 1L,
        email: String = "test@example.com",
        name: String = "Test User",
        active: Boolean = true
    ) = User(id = id, email = email, name = name, active = active)

    fun aCreateUserRequest(
        email: String = "test@example.com",
        name: String = "Test User"
    ) = CreateUserRequest(email = email, name = name)
}

// Usage
val user = UserTestFixtures.aUser(email = "custom@example.com")
val inactive = UserTestFixtures.aUser(active = false)
```

## Coroutine Testing

```kotlin
import kotlinx.coroutines.test.runTest
import kotlinx.coroutines.test.StandardTestDispatcher

@ExtendWith(MockKExtension::class)
class UserCoroutineServiceTest {

    @MockK
    lateinit var repository: UserRepository

    @InjectMockKs
    lateinit var service: UserCoroutineService

    @Test
    fun `findById - suspend - returns user`() = runTest {
        coEvery { repository.findById(1L) } returns Optional.of(User(id = 1L))

        val result = service.findById(1L)

        assertThat(result).isNotNull
        coVerify { repository.findById(1L) }
    }
}
```

## Quick Reference

| Annotation / Function | Purpose |
|----------------------|---------|
| `@ExtendWith(MockKExtension::class)` | Enable MockK in JUnit 5 |
| `@MockK` | Create mock |
| `@InjectMockKs` | Inject mocks into subject under test |
| `@MockkBean` | Mock Spring bean in `@WebMvcTest` / `@SpringBootTest` |
| `every { } returns` | Stub method call |
| `coEvery { } returns` | Stub suspend function |
| `verify { }` | Verify interaction |
| `coVerify { }` | Verify suspend interaction |
| `confirmVerified()` | Assert no unexpected interactions |
| `every { } throws` | Stub exception |
| `runTest { }` | Run coroutine test |
| `@WithMockUser` | Mock authenticated user for security tests |
| `assertThat()` | AssertJ fluent assertions |
