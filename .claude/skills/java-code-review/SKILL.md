---
name: java-code-review
description: Systematic code review for Kotlin/Spring Boot with null safety, !! operator detection, data class JPA misuse, @field: annotation targets, coroutine usage, and performance checks. Use when user says "review code", "check this PR", "code review", or before merging changes.
---

# Kotlin Code Review Skill

Systematic code review checklist for Kotlin/Spring Boot projects.

## When to Use
- User says "review this code" / "check this PR" / "code review"
- Before merging a PR
- After implementing a feature

## Review Strategy

1. **Quick scan** - Understand intent, identify scope
2. **Checklist pass** - Go through each category below
3. **Summary** - List findings by severity (Critical → Minor)

## Output Format

```markdown
## Code Review: [file/feature name]

### Critical
- [Issue description + line reference + suggestion]

### Improvements
- [Suggestion + rationale]

### Minor/Style
- [Nitpicks, optional improvements]

### Good Practices Observed
- [Positive feedback - important for morale]
```

---

## Review Checklist

### 1. Kotlin Null Safety

**Check for `!!` operator usage:**
```kotlin
// ❌ CRITICAL: !! can throw NullPointerException at runtime
val name = user.name!!.uppercase()

// ✅ Safe alternatives
val name = user.name?.uppercase() ?: ""           // Elvis operator
val name = user.name?.uppercase() ?: return       // early return
val name = requireNotNull(user.name) { "name must not be null" }  // explicit error
```

**Flags:**
- Any `!!` usage — always flag as Critical or High
- Chained `?.` that eventually force-unwrap
- `Optional.get()` without `isPresent()` check

**Suggest:**
- Use `?.` (safe call) + `?:` (Elvis) for nullable values
- Use `let`, `run`, `also` for scoped null checks
- Return early or throw descriptive exception instead of `!!`

### 2. JPA Entity Design (Kotlin-specific)

**Check for `data class` used as JPA entity:**
```kotlin
// ❌ CRITICAL: data class breaks Hibernate proxies
@Entity
data class User(
    @Id @GeneratedValue val id: Long = 0,
    var email: String = ""
)

// ✅ Must use open class
@Entity
open class User(
    @Id @GeneratedValue open val id: Long = 0,
    open var email: String = ""
)
```

**Check for missing `@field:` targets:**
```kotlin
// ❌ HIGH: validation annotation silently ignored
data class CreateUserRequest(
    @NotBlank val email: String  // ← target is PARAMETER, not FIELD → never validated
)

// ✅ @field: target ensures validation runs
data class CreateUserRequest(
    @field:NotBlank val email: String
)
```

**Flags:**
- `data class` with `@Entity` annotation — Critical
- Validation annotations on data class params without `@field:` — High
- `toString()` in entity that includes lazy collections — High (triggers lazy load)
- Missing `override` on entity properties when using inheritance

### 3. Coroutine Usage

**Check for blocking calls in suspend context:**
```kotlin
// ❌ HIGH: blocking in coroutine starves thread pool
suspend fun getUser(id: Long): User {
    Thread.sleep(1000)              // ← blocks coroutine thread
    return repository.findById(id).orElseThrow()  // ← JPA is blocking
}

// ✅ Use Dispatchers.IO for blocking operations
suspend fun getUser(id: Long): User = withContext(Dispatchers.IO) {
    repository.findById(id).orElseThrow()
}
```

**Check for `.block()` calls inside coroutines or WebFlux chains:**
```kotlin
// ❌ CRITICAL: .block() inside reactive context causes deadlock
fun Mono<User>.getBlocking(): User = this.block()!!  // never do this in reactive context
```

**Flags:**
- `Thread.sleep()`, `Thread.join()` inside `suspend fun`
- `.block()` called inside coroutine or reactive context
- Missing `withContext(Dispatchers.IO)` around JPA/JDBC calls
- Coroutine scope leaks (not using `coroutineScope { }` or structured concurrency)
- `runBlocking` in production code (acceptable only in tests and main)

### 4. Scope Functions Misuse

```kotlin
// ❌ MEDIUM: chained scope functions reduce readability
val result = user
    .let { it.copy() }
    .also { log.info("user: $it") }
    .run { this.email.uppercase() }
    .let { email -> "$email processed" }

// ✅ Flat, readable code
log.info("user: $user")
val result = "${user.email.uppercase()} processed"

// ✅ Appropriate let usage — single nullable chain
val name = user?.name?.let { it.trim().takeIf { it.isNotEmpty() } } ?: "Anonymous"
```

**Scope function guidelines:**
| Function | Use when |
|----------|---------|
| `let` | Nullable check, transform result |
| `apply` | Configure object after creation |
| `also` | Side effects (logging) without transforming |
| `run` | Execute block with receiver, return result |
| `with` | Multiple operations on non-null receiver |

### 5. Exception Handling

```kotlin
// ❌ Swallowing exceptions
try {
    process()
} catch (e: Exception) {
    // silently ignored — Critical
}

// ❌ Losing stack trace
catch (e: IOException) {
    throw RuntimeException(e.message)  // ← loses original stack trace
}

// ✅ Proper handling
catch (e: IOException) {
    log.error("Failed to process file: {}", filename, e)
    throw ProcessingException("File processing failed", e)
}
```

**Kotlin-specific flags:**
- Catching `Exception` or `Throwable` too broadly
- Throwing exceptions from coroutines without propagating to CoroutineExceptionHandler
- Using exceptions for flow control in coroutines (prefer sealed class Result)

### 6. Collections & Immutability

```kotlin
// ❌ Mutating while iterating
for (item in items) {
    if (item.isExpired()) items.remove(item)  // ConcurrentModificationException
}

// ✅ Use removeIf or filter
items.removeIf { it.isExpired() }
val activeItems = items.filter { !it.isExpired() }

// ❌ Exposing mutable collection from API
class Service {
    private val _items = mutableListOf<Item>()
    fun getItems(): MutableList<Item> = _items  // ← caller can mutate
}

// ✅ Return immutable view
fun getItems(): List<Item> = _items.toList()
```

**Flags:**
- Modifying collections during iteration
- Returning `MutableList`/`MutableMap` from public APIs
- `toMutableList()` on results of `findAll()` when mutation not needed

### 7. Spring-Specific Patterns

```kotlin
// ❌ Field injection (breaks testability)
@Service
class UserService {
    @Autowired
    private lateinit var userRepository: UserRepository
}

// ✅ Constructor injection
@Service
class UserService(private val userRepository: UserRepository)

// ❌ Missing @Transactional on write operations
@Service
class OrderService(private val repo: OrderRepository) {
    fun createOrder(request: CreateOrderRequest): Order {
        val order = repo.save(Order(...))
        sendNotification(order)  // if this throws, save is NOT rolled back
        return order
    }
}

// ✅ @Transactional wraps both operations
@Transactional
fun createOrder(request: CreateOrderRequest): Order { ... }
```

**Flags:**
- Field injection with `@Autowired` — use constructor injection
- Missing `@Transactional` on multi-step write operations
- `@Transactional` on private methods (Spring AOP cannot intercept)
- `@Transactional` self-invocation (calling from same class bypasses AOP)

### 8. Concurrency & Thread Safety

```kotlin
// ❌ Non-thread-safe mutable state in singleton
@Service
class CacheService {
    private val cache = HashMap<String, User>()  // ← not thread-safe
}

// ✅ Thread-safe
@Service
class CacheService {
    private val cache = ConcurrentHashMap<String, User>()
}

// ✅ Or use atomic operations
private val count = AtomicLong(0)
```

### 9. Performance

```kotlin
// ❌ String concatenation in loop
var result = ""
for (s in strings) { result += s }  // O(n²)

// ✅ StringBuilder or joinToString
val result = strings.joinToString("")

// ❌ N+1 in loop
for (user in users) {
    val orders = orderRepo.findByUserId(user.id)  // N queries!
}

// ✅ Fetch all at once
val ordersByUser = orderRepo.findByUserIdIn(userIds).groupBy { it.userId }
```

### 10. Data Class Design

```kotlin
// ❌ data class with too many fields (violation of single responsibility)
data class UserResponse(
    val id: Long, val name: String, val email: String,
    val address: String, val city: String, val country: String,
    val orderCount: Int, val lastLoginAt: LocalDateTime,
    val createdAt: LocalDateTime, val roles: List<String>
)

// ✅ Nested data classes for logical grouping
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String,
    val address: AddressResponse,
    val stats: UserStatsResponse
)
```

---

## Severity Guidelines

| Severity | Criteria |
|----------|----------|
| **Critical** | `!!` operator, `data class` as JPA entity, security vulnerability, data loss risk |
| **High** | Missing `@field:` validation, blocking in coroutine, N+1 query, `@Transactional` missing |
| **Medium** | Scope function misuse, exception swallowing, non-thread-safe shared state |
| **Low** | Style, naming, minor optimization, missing KDoc on public API |

## Quick Reference Card

| Category | Key Checks |
|----------|------------|
| Null Safety | `!!` usage, unhandled nullable, `?.let { }` patterns |
| JPA Kotlin | `data class @Entity`, missing `@field:`, lazy `toString()` |
| Coroutines | Blocking in `suspend`, `.block()`, missing `Dispatchers.IO` |
| Scope Functions | Overuse of `let`/`run`/`also`/`apply`, readability |
| Exceptions | Empty catch, broad catch, lost stack trace |
| Collections | Mutation during iteration, exposing mutable from API |
| Spring | Field injection, `@Transactional` on private/self-invoked |
| Performance | String concat in loop, N+1 queries |
