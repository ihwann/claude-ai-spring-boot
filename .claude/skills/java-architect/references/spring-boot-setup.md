# Spring Boot Setup (Kotlin)

## Project Structure (Clean Architecture)

```
src/main/kotlin/pl/piomin/services/
├── domain/              # Core business logic
│   ├── model/          # Entities (open class), value objects
│   ├── repository/     # Repository interfaces
│   └── service/        # Domain services
├── application/         # Use cases
│   ├── dto/            # Request/Response DTOs (data class)
│   ├── mapper/         # Entity <-> DTO mappers / companion factories
│   └── service/        # Application services
├── infrastructure/      # External concerns
│   ├── persistence/    # JPA implementations
│   ├── config/         # Spring configuration
│   └── security/       # Security setup
└── presentation/        # API layer
    └── rest/           # REST controllers
```

## Kotlin pom.xml (Spring Boot 3.x)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.0</version>
    </parent>

    <groupId>pl.piomin.services</groupId>
    <artifactId>my-service</artifactId>
    <version>1.0.0</version>

    <properties>
        <java.version>21</java.version>
        <kotlin.version>2.1.0</kotlin.version>
        <kotlin.compiler.incremental>true</kotlin.compiler.incremental>
        <mockk.version>1.13.14</mockk.version>
        <springmockk.version>4.0.2</springmockk.version>
    </properties>

    <dependencies>
        <!-- Kotlin -->
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-stdlib</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-reflect</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.module</groupId>
            <artifactId>jackson-module-kotlin</artifactId>
        </dependency>

        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
        </dependency>
        <dependency>
            <groupId>org.flywaydb</groupId>
            <artifactId>flyway-core</artifactId>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.mockito</groupId>
                    <artifactId>mockito-core</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>io.mockk</groupId>
            <artifactId>mockk-jvm</artifactId>
            <version>${mockk.version}</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>com.ninja-squad</groupId>
            <artifactId>springmockk</artifactId>
            <version>${springmockk.version}</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <sourceDirectory>${project.basedir}/src/main/kotlin</sourceDirectory>
        <testSourceDirectory>${project.basedir}/src/test/kotlin</testSourceDirectory>

        <plugins>
            <!-- Kotlin compiler — must run before maven-compiler-plugin -->
            <plugin>
                <groupId>org.jetbrains.kotlin</groupId>
                <artifactId>kotlin-maven-plugin</artifactId>
                <version>${kotlin.version}</version>
                <configuration>
                    <args>
                        <arg>-Xjsr305=strict</arg>
                    </args>
                    <compilerPlugins>
                        <!-- all-open: makes @Component, @Service, @Entity etc. open -->
                        <plugin>spring</plugin>
                        <!-- no-arg: zero-arg constructor for @Entity, @Embeddable -->
                        <plugin>jpa</plugin>
                    </compilerPlugins>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>org.jetbrains.kotlin</groupId>
                        <artifactId>kotlin-maven-allopen</artifactId>
                        <version>${kotlin.version}</version>
                    </dependency>
                    <dependency>
                        <groupId>org.jetbrains.kotlin</groupId>
                        <artifactId>kotlin-maven-noarg</artifactId>
                        <version>${kotlin.version}</version>
                    </dependency>
                </dependencies>
                <executions>
                    <execution>
                        <id>compile</id>
                        <goals><goal>compile</goal></goals>
                        <configuration>
                            <sourceDirs>
                                <sourceDir>${project.basedir}/src/main/kotlin</sourceDir>
                            </sourceDirs>
                        </configuration>
                    </execution>
                    <execution>
                        <id>test-compile</id>
                        <goals><goal>test-compile</goal></goals>
                        <configuration>
                            <sourceDirs>
                                <sourceDir>${project.basedir}/src/test/kotlin</sourceDir>
                            </sourceDirs>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

            <!-- Disable default maven-compiler-plugin phases -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <executions>
                    <execution>
                        <id>default-compile</id>
                        <phase>none</phase>
                    </execution>
                    <execution>
                        <id>default-testCompile</id>
                        <phase>none</phase>
                    </execution>
                </executions>
            </plugin>

            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

## Application Configuration

```yaml
# application.yml
spring:
  application:
    name: my-service

  datasource:
    url: ${DATABASE_URL:jdbc:postgresql://localhost:5432/mydb}
    username: ${DATABASE_USER:myuser}
    password: ${DATABASE_PASSWORD:mypassword}
    hikari:
      maximum-pool-size: 10
      minimum-idle: 5
      connection-timeout: 20000

  jpa:
    hibernate:
      ddl-auto: validate
    open-in-view: false     # Always disable — prevents lazy loading outside transaction
    properties:
      hibernate:
        jdbc:
          batch_size: 20
        order_inserts: true
        order_updates: true

  flyway:
    enabled: true
    baseline-on-migrate: true
    locations: classpath:db/migration

server:
  port: 8080
  shutdown: graceful

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when-authorized
```

## Main Application Class

```kotlin
package pl.piomin.services

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication
import org.springframework.data.jpa.repository.config.EnableJpaAuditing

@SpringBootApplication
@EnableJpaAuditing
class MyServiceApplication  // No main function in the class — use top-level function

fun main(args: Array<String>) {
    runApplication<MyServiceApplication>(*args)
}
```

## Configuration Classes (Kotlin style)

```kotlin
package pl.piomin.services.infrastructure.config

import io.swagger.v3.oas.models.OpenAPI
import io.swagger.v3.oas.models.info.Info
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class OpenApiConfig {

    @Bean
    fun customOpenAPI(): OpenAPI = OpenAPI()
        .info(
            Info()
                .title("My Service API")
                .version("1.0.0")
                .description("Enterprise Kotlin microservice API")
        )
}
```

## Exception Handling (ProblemDetail)

```kotlin
package pl.piomin.services.infrastructure.config

import org.springframework.http.HttpStatus
import org.springframework.http.ProblemDetail
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.RestControllerAdvice
import java.time.Instant

@RestControllerAdvice
class GlobalExceptionHandler {

    @ExceptionHandler(EntityNotFoundException::class)
    fun handleNotFound(ex: EntityNotFoundException): ProblemDetail =
        ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.message ?: "Not found")
            .also { it.setProperty("timestamp", Instant.now()) }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    fun handleValidation(ex: MethodArgumentNotValidException): ProblemDetail =
        ProblemDetail.forStatusAndDetail(HttpStatus.BAD_REQUEST, "Validation failed").also { problem ->
            problem.setProperty("errors", ex.bindingResult.fieldErrors
                .map { "${it.field}: ${it.defaultMessage}" })
        }
}
```

## Quick Reference

| Component | Kotlin Pattern |
|-----------|--------------|
| `@SpringBootApplication` | Main class (no `main` inside) + top-level `fun main` |
| `@Configuration` | Regular class |
| `@Bean` | Extension or method |
| `@Value` | Constructor param with default |
| `@ConfigurationProperties` | `data class` preferred |
| `@Profile` | Environment-specific beans |
| `@EnableJpaAuditing` | Automatic audit fields |
| `ProblemDetail` | RFC 7807 error responses |
| `runApplication<App>(*args)` | Entry point — Kotlin idiomatic |
