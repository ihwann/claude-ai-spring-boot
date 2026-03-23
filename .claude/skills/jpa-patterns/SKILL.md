---
name: jpa-patterns
description: JPA/Hibernate patterns and common pitfalls for Kotlin (N+1, lazy loading, transactions, queries, open class entities). Use when user has JPA performance issues, LazyInitializationException, or asks about entity relationships, fetching strategies, or Kotlin-specific JPA issues.
---

# JPA Patterns Skill (Kotlin)

Best practices and common pitfalls for JPA/Hibernate in Kotlin/Spring applications.

## When to Use
- User mentions "N+1 problem" / "too many queries"
- LazyInitializationException errors
- Questions about fetch strategies (EAGER vs LAZY)
- Transaction management issues
- Entity relationship design
- Query optimization
- Kotlin-specific JPA issues (open class, @field:, data class)

---

## Quick Reference: Common Problems

| Problem | Symptom | Solution |
|---------|---------|----------|
| N+1 queries | Many SELECT statements | JOIN FETCH, @EntityGraph |
| LazyInitializationException | Error outside transaction | JOIN FETCH, @Transactional, DTO projection |
| Slow queries | Performance issues | Pagination, projections, indexes |
| Dirty checking overhead | Slow updates | Read-only transactions, DTOs |
| Lost updates | Concurrent modifications | Optimistic locking (@Version) |
| Hibernate proxy error | ClassCastException | Use open class, not data class |
| Validation ignored | No constraint violations triggered | Use @field: annotation target |

---

## Kotlin JPA Essentials

> **Read this first before any other section**

### ❌ data class for JPA entities — NEVER DO THIS

```kotlin
// ❌ WRONG: data class breaks Hibernate
@Entity
data class User(
    @Id @GeneratedValue val id: Long = 0,
    var email: String = ""
)
// Problems:
// 1. Kotlin data classes are final — Hibernate can't create proxies
// 2. auto-generated equals/hashCode uses all fields — breaks with detached entities
// 3. copy() bypasses JPA lifecycle callbacks
```

### ✅ open class for JPA entities

```kotlin
// ✅ CORRECT: open class for Hibernate proxy support
// (all-open Maven plugin with 'jpa' preset handles @Entity automatically)
@Entity
@Table(name = "users")
open class User(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    open val id: Long = 0,

    open var email: String = "",

    open var active: Boolean = true
) {
    // Implement equals/hashCode using business key or id
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is User) return false
        return id != 0L && id == other.id
    }
    override fun hashCode(): Int = id.hashCode()
    override fun toString(): String = "User(id=$id, email=$email)"
}
```

### @field: annotation target — REQUIRED for validation

```kotlin
// ❌ WRONG: validation annotation is on constructor parameter, not the field
data class CreateUserRequest(
    @NotBlank val email: String  // annotation target is parameter — validation IGNORED
)

// ✅ CORRECT: @field: target ensures annotation is placed on the backing field
data class CreateUserRequest(
    @field:NotBlank(message = "Email is required")
    @field:Email(message = "Invalid email format")
    val email: String,

    @field:NotNull
    @field:Min(18)
    val age: Int
)
```

### MutableList vs List for collections

```kotlin
@Entity
open class Author(
    @Id @GeneratedValue open val id: Long = 0,
    open var name: String = "",

    // Use MutableList for bidirectional associations (Hibernate modifies the collection)
    @OneToMany(mappedBy = "author", cascade = [CascadeType.ALL], orphanRemoval = true)
    open val books: MutableList<Book> = mutableListOf()
) {
    fun addBook(book: Book) {
        books.add(book)
        book.author = this
    }

    fun removeBook(book: Book) {
        books.remove(book)
        book.author = null
    }
}
```

---

## N+1 Problem

> The #1 JPA performance killer

### The Problem

```kotlin
// ❌ BAD: N+1 queries
val authors = authorRepository.findAll()  // 1 query
for (author in authors) {
    println(author.books.size)  // N queries! (books is LAZY)
}
// 100 authors = 101 queries
```

### Solution 1: JOIN FETCH (JPQL)

```kotlin
// ✅ GOOD: Single query with JOIN FETCH
interface AuthorRepository : JpaRepository<Author, Long> {

    @Query("SELECT a FROM Author a JOIN FETCH a.books")
    fun findAllWithBooks(): List<Author>
}
```

### Solution 2: @EntityGraph

```kotlin
// ✅ GOOD: EntityGraph for declarative fetching
interface AuthorRepository : JpaRepository<Author, Long> {

    @EntityGraph(attributePaths = ["books"])
    override fun findAll(): List<Author>

    // Or with named graph defined on entity
    @EntityGraph(value = "Author.withBooks")
    fun findAllWithBooks(): List<Author>
}

// Named graph on entity
@Entity
@NamedEntityGraph(
    name = "Author.withBooks",
    attributeNodes = [NamedAttributeNode("books")]
)
open class Author(...)
```

### Solution 3: Batch Fetching

```kotlin
// ✅ GOOD: Hibernate batch fetching
@Entity
open class Author(
    @OneToMany(mappedBy = "author")
    @BatchSize(size = 25)
    open val books: MutableList<Book> = mutableListOf()
)

// Or globally in application.yml
// spring.jpa.properties.hibernate.default_batch_fetch_size=25
```

### Detecting N+1

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true
logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

---

## Lazy Loading

### Best Practice: Default to LAZY

```kotlin
@Entity
open class Order(
    // ✅ LAZY for @ManyToOne — override the EAGER default
    @ManyToOne(fetch = FetchType.LAZY)
    open var customer: Customer? = null,

    // LAZY is default for @OneToMany
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    open val items: MutableList<OrderItem> = mutableListOf()
)
```

### LazyInitializationException Solutions

**Solution 1: JOIN FETCH in query**
```kotlin
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
fun findByIdWithItems(@Param("id") id: Long): Optional<Order>
```

**Solution 2: @Transactional on service method**
```kotlin
@Service
class OrderService(private val orderRepository: OrderRepository) {

    @Transactional(readOnly = true)
    fun getOrderWithItems(id: Long): OrderDto {
        val order = orderRepository.findById(id).orElseThrow()
        val itemCount = order.items.size  // safe — within transaction
        return OrderDto(order.id, itemCount)
    }
}
```

**Solution 3: DTO Projection (recommended)**
```kotlin
// Interface projection
interface OrderSummary {
    val id: Long
    val status: String
    val itemCount: Int
}

@Query("SELECT o.id as id, o.status as status, SIZE(o.items) as itemCount FROM Order o WHERE o.id = :id")
fun findOrderSummary(@Param("id") id: Long): Optional<OrderSummary>
```

---

## Transactions

```kotlin
@Service
@Transactional(readOnly = true)  // Default read-only for all methods
class OrderService(private val orderRepository: OrderRepository) {

    fun findById(id: Long): Order =
        orderRepository.findById(id).orElseThrow { EntityNotFoundException("Order $id not found") }

    @Transactional  // Write transaction
    fun create(request: CreateOrderRequest): Order =
        orderRepository.save(Order(customerId = request.customerId))

    @Transactional(rollbackFor = [Exception::class])
    fun processPayment(orderId: Long) {
        // Rolls back on any exception
    }
}
```

### Transaction Propagation

```kotlin
@Service
class OrderService(
    private val orderRepository: OrderRepository,
    private val paymentService: PaymentService
) {
    @Transactional
    fun placeOrder(order: Order) {
        orderRepository.save(order)
        paymentService.processPayment(order)  // REQUIRED: uses existing transaction
    }
}

@Service
class PaymentService {
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // Always new transaction
    fun processPayment(order: Order) { /* independent */ }
}
```

---

## Entity Relationships

### OneToMany / ManyToOne

```kotlin
@Entity
open class Author(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) open val id: Long = 0,
    open var name: String = "",

    @OneToMany(mappedBy = "author", cascade = [CascadeType.ALL], orphanRemoval = true)
    open val books: MutableList<Book> = mutableListOf()
) {
    fun addBook(book: Book) { books.add(book); book.author = this }
    fun removeBook(book: Book) { books.remove(book); book.author = null }
}

@Entity
open class Book(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) open val id: Long = 0,
    open var title: String = "",

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    open var author: Author? = null
)
```

### equals/hashCode for JPA Entities

```kotlin
// ✅ Business-key based equals/hashCode
@Entity
open class Book(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) open val id: Long = 0,

    @NaturalId
    @Column(unique = true, nullable = false)
    open var isbn: String = ""
) {
    override fun equals(other: Any?): Boolean {
        if (this === other) return true
        if (other !is Book) return false
        return isbn == other.isbn
    }
    override fun hashCode(): Int = isbn.hashCode()
    override fun toString(): String = "Book(isbn=$isbn)"  // Never include lazy fields!
}
```

---

## Query Optimization

### Pagination

```kotlin
interface OrderRepository : JpaRepository<Order, Long> {
    fun findByStatus(status: OrderStatus, pageable: Pageable): Page<Order>
}

// Usage
val pageable = PageRequest.of(0, 20, Sort.by("createdAt").descending())
val orders = orderRepository.findByStatus(OrderStatus.PENDING, pageable)
```

### DTO Projections

```kotlin
// Class-based projection (data class)
data class OrderSummary(val id: Long, val customerName: String, val total: BigDecimal)

@Query("SELECT new pl.piomin.services.dto.OrderSummary(o.id, o.customer.name, o.total) FROM Order o WHERE o.status = :status")
fun findOrderSummaries(@Param("status") status: OrderStatus): List<OrderSummary>
```

### Bulk Operations

```kotlin
interface OrderRepository : JpaRepository<Order, Long> {

    @Modifying
    @Query("UPDATE Order o SET o.status = :status WHERE o.createdAt < :date")
    fun updateOldOrdersStatus(@Param("status") status: OrderStatus, @Param("date") date: LocalDateTime): Int

    @Modifying
    @Query("DELETE FROM Order o WHERE o.status = :status AND o.createdAt < :date")
    fun deleteOldOrders(@Param("status") status: OrderStatus, @Param("date") date: LocalDateTime): Int
}
```

---

## Optimistic Locking

```kotlin
@Entity
open class Order(
    @Id @GeneratedValue open val id: Long = 0,

    @Version
    open var version: Long = 0,  // Hibernate manages this automatically

    open var status: OrderStatus = OrderStatus.PENDING
)

// Handle concurrent modification
@Service
class OrderService(private val orderRepository: OrderRepository) {

    @Transactional
    fun updateOrder(id: Long, request: UpdateOrderRequest): Order {
        return try {
            val order = orderRepository.findById(id).orElseThrow()
            order.status = request.status
            orderRepository.save(order)
        } catch (e: OptimisticLockException) {
            throw ConcurrentModificationException("Order was modified concurrently. Please retry.")
        }
    }
}
```

---

## Performance Checklist

- [ ] No `data class` for JPA entities — use `open class`
- [ ] `@field:` target on all validation annotations in data class DTOs
- [ ] No N+1 queries (use JOIN FETCH or @EntityGraph)
- [ ] LAZY fetch by default (especially @ManyToOne)
- [ ] Pagination for large result sets
- [ ] DTO projections for read-only queries
- [ ] Bulk operations for batch updates/deletes
- [ ] @Version for entities with concurrent access
- [ ] Indexes on frequently queried columns
- [ ] No lazy fields in toString()
- [ ] Read-only transactions (`@Transactional(readOnly = true)`) where applicable
- [ ] `open-in-view: false` in application.yml

---

## Related Skills

- `spring-boot-patterns` - Kotlin Spring Boot controller/service patterns
- `java-code-review` - Kotlin code review checklist
- `clean-code` - Code quality principles
