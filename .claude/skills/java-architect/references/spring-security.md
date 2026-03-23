# Spring Security (Kotlin DSL)

> Always use the Kotlin DSL (`http { }`) for Spring Security configuration.
> Avoid the Java-style lambda configuration in Kotlin projects.

## Security Configuration (Kotlin DSL)

```kotlin
package pl.piomin.services.infrastructure.security

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.http.HttpMethod
import org.springframework.security.authentication.AuthenticationManager
import org.springframework.security.authentication.dao.DaoAuthenticationProvider
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity
import org.springframework.security.config.annotation.web.builders.HttpSecurity
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity
import org.springframework.security.config.http.SessionCreationPolicy
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder
import org.springframework.security.crypto.password.PasswordEncoder
import org.springframework.security.web.SecurityFilterChain
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(securedEnabled = true, jsr250Enabled = true)
class SecurityConfig(
    private val jwtAuthFilter: JwtAuthenticationFilter,
    private val userDetailsService: CustomUserDetailsService
) {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain = http {
        csrf { disable() }
        cors { }
        sessionManagement {
            sessionCreationPolicy = SessionCreationPolicy.STATELESS
        }
        authorizeHttpRequests {
            authorize("/api/auth/**", permitAll)
            authorize("/api/public/**", permitAll)
            authorize("/actuator/health", permitAll)
            authorize("/actuator/info", permitAll)
            authorize("/swagger-ui/**", permitAll)
            authorize("/v3/api-docs/**", permitAll)
            authorize(HttpMethod.GET, "/api/products/**", permitAll)
            authorize("/api/admin/**", hasRole("ADMIN"))
            authorize(anyRequest, authenticated)
        }
        addFilterBefore<UsernamePasswordAuthenticationFilter>(jwtAuthFilter)
        authenticationProvider(authenticationProvider())
    }.build()

    @Bean
    fun authenticationProvider(): DaoAuthenticationProvider =
        DaoAuthenticationProvider().apply {
            setUserDetailsService(userDetailsService)
            setPasswordEncoder(passwordEncoder())
        }

    @Bean
    fun authenticationManager(config: AuthenticationConfiguration): AuthenticationManager =
        config.authenticationManager

    @Bean
    fun passwordEncoder(): PasswordEncoder = BCryptPasswordEncoder(12)
}
```

## OAuth2 Resource Server (JWT)

```kotlin
@Configuration
@EnableMethodSecurity
class OAuth2SecurityConfig {

    @Bean
    fun filterChain(http: HttpSecurity): SecurityFilterChain = http {
        csrf { disable() }
        sessionManagement {
            sessionCreationPolicy = SessionCreationPolicy.STATELESS
        }
        authorizeHttpRequests {
            authorize("/actuator/health", permitAll)
            authorize("/api/public/**", permitAll)
            authorize(anyRequest, authenticated)
        }
        oauth2ResourceServer {
            jwt {
                // Optionally customize JWT decoder
            }
        }
    }.build()
}
```

## JWT Authentication Filter

```kotlin
package pl.piomin.services.infrastructure.security

import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter

@Component
class JwtAuthenticationFilter(
    private val jwtService: JwtService,
    private val userDetailsService: CustomUserDetailsService
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain
    ) {
        val authHeader = request.getHeader("Authorization")

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response)
            return
        }

        val jwt = authHeader.substring(7)
        val userEmail = jwtService.extractUsername(jwt) ?: run {
            filterChain.doFilter(request, response)
            return
        }

        if (SecurityContextHolder.getContext().authentication == null) {
            val userDetails = userDetailsService.loadUserByUsername(userEmail)
            if (jwtService.isTokenValid(jwt, userDetails)) {
                val authToken = UsernamePasswordAuthenticationToken(
                    userDetails,
                    null,
                    userDetails.authorities
                ).also {
                    it.details = WebAuthenticationDetailsSource().buildDetails(request)
                }
                SecurityContextHolder.getContext().authentication = authToken
            }
        }

        filterChain.doFilter(request, response)
    }
}
```

## JWT Service

```kotlin
package pl.piomin.services.infrastructure.security

import io.jsonwebtoken.Claims
import io.jsonwebtoken.Jwts
import io.jsonwebtoken.io.Decoders
import io.jsonwebtoken.security.Keys
import org.springframework.beans.factory.annotation.Value
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.stereotype.Service
import java.security.Key
import java.util.*
import java.util.function.Function

@Service
class JwtService(
    @Value("\${jwt.secret}") private val secretKey: String,
    @Value("\${jwt.expiration}") private val jwtExpiration: Long
) {
    fun extractUsername(token: String): String? =
        extractClaim(token, Claims::getSubject)

    fun <T> extractClaim(token: String, claimsResolver: Function<Claims, T>): T =
        claimsResolver.apply(extractAllClaims(token))

    fun generateToken(userDetails: UserDetails): String =
        buildToken(emptyMap(), userDetails, jwtExpiration)

    private fun buildToken(
        extraClaims: Map<String, Any>,
        userDetails: UserDetails,
        expiration: Long
    ): String = Jwts.builder()
        .claims(extraClaims)
        .subject(userDetails.username)
        .issuedAt(Date())
        .expiration(Date(System.currentTimeMillis() + expiration))
        .signWith(getSignInKey())
        .compact()

    fun isTokenValid(token: String, userDetails: UserDetails): Boolean {
        val username = extractUsername(token)
        return username == userDetails.username && !isTokenExpired(token)
    }

    private fun isTokenExpired(token: String): Boolean =
        extractClaim(token, Claims::getExpiration).before(Date())

    private fun extractAllClaims(token: String): Claims =
        Jwts.parser()
            .verifyWith(getSignInKey() as javax.crypto.SecretKey)
            .build()
            .parseSignedClaims(token)
            .payload

    private fun getSignInKey(): Key =
        Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey))
}
```

## UserDetailsService

```kotlin
package pl.piomin.services.infrastructure.security

import org.springframework.security.core.authority.SimpleGrantedAuthority
import org.springframework.security.core.userdetails.User
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.security.core.userdetails.UserDetailsService
import org.springframework.security.core.userdetails.UsernameNotFoundException
import org.springframework.stereotype.Service

@Service
class CustomUserDetailsService(
    private val userRepository: UserRepository
) : UserDetailsService {

    override fun loadUserByUsername(email: String): UserDetails =
        userRepository.findByEmail(email)
            .map { user ->
                User.builder()
                    .username(user.email)
                    .password(user.password)
                    .authorities(user.roles.map { SimpleGrantedAuthority(it.name) })
                    .accountLocked(!user.active)
                    .build()
            }
            .orElseThrow { UsernameNotFoundException("User not found: $email") }
}
```

## Method-Level Security

```kotlin
package pl.piomin.services.application.service

import org.springframework.security.access.prepost.PostAuthorize
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.stereotype.Service

@Service
class UserService(private val userRepository: UserRepository) {

    @PreAuthorize("hasRole('ADMIN')")
    fun getAllUsers(): List<User> = userRepository.findAll()

    @PreAuthorize("hasRole('USER') or hasRole('ADMIN')")
    fun getUserById(id: Long): User =
        userRepository.findById(id)
            .orElseThrow { EntityNotFoundException("User not found") }

    @PreAuthorize("#user.id == authentication.principal.id or hasRole('ADMIN')")
    fun updateUser(user: User): User = userRepository.save(user)

    @PostAuthorize("returnObject.email == authentication.principal.username or hasRole('ADMIN')")
    fun findUserByEmail(email: String): User =
        userRepository.findByEmail(email)
            .orElseThrow { EntityNotFoundException("User not found") }

    @PreAuthorize("@userSecurityService.hasAccess(#userId)")
    fun deleteUser(userId: Long) = userRepository.deleteById(userId)
}
```

## Security Utilities (Kotlin)

```kotlin
package pl.piomin.services.infrastructure.security

import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.security.core.userdetails.UserDetails
import org.springframework.stereotype.Component

@Component("userSecurityService")
class UserSecurityService {

    fun hasAccess(userId: Long): Boolean {
        val auth = SecurityContextHolder.getContext().authentication
        return auth?.isAuthenticated == true
    }

    fun getCurrentUsername(): String? =
        (SecurityContextHolder.getContext().authentication?.principal as? UserDetails)?.username

    fun hasRole(role: String): Boolean =
        SecurityContextHolder.getContext().authentication
            ?.authorities
            ?.any { it.authority == "ROLE_$role" } == true
}
```

## CORS Configuration

```kotlin
@Configuration
class CorsConfig {
    @Bean
    fun corsConfigurationSource(): CorsConfigurationSource =
        UrlBasedCorsConfigurationSource().apply {
            registerCorsConfiguration("/**", CorsConfiguration().apply {
                allowedOrigins = listOf("http://localhost:3000")
                allowedMethods = listOf("GET", "POST", "PUT", "DELETE", "OPTIONS")
                allowedHeaders = listOf("*")
                allowCredentials = true
            })
        }
}
```

## Configuration Properties

```yaml
jwt:
  secret: ${JWT_SECRET:your-256-bit-secret-change-in-production}
  expiration: 86400000   # 24 hours

spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://accounts.example.com
          jwk-set-uri: https://accounts.example.com/.well-known/jwks.json
```

## Quick Reference

| Annotation | Purpose |
|-----------|---------|
| `@EnableWebSecurity` | Enable Spring Security |
| `@EnableMethodSecurity` | Enable `@PreAuthorize`, `@PostAuthorize` |
| `http { ... }` | Kotlin DSL for SecurityFilterChain |
| `authorize(anyRequest, authenticated)` | Kotlin DSL authorization rule |
| `@PreAuthorize("hasRole('ADMIN')")` | Method-level access control |
| `@PostAuthorize` | Check after method execution |
| `@AuthenticationPrincipal` | Inject current user into controller |
| `SecurityContextHolder` | Access current security context |
