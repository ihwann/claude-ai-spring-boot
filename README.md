# Claude Code Template for Kotlin/Spring Boot Application

This template provides a structured starting point for **Kotlin/Spring Boot** applications, optimized for Claude AI's code completion capabilities. It includes essential configurations, Kotlin-specific best practices, and pre-configured agents and skills to streamline development.

Clone this repository and use it with Claude Code to generate your Kotlin/Spring Boot application.

## Tech Stack

| Component | Technology |
|-----------|------------|
| Language | **Kotlin 2.x** |
| Framework | Spring Boot 3.x |
| Build | Maven (kotlin-maven-plugin) |
| JVM | Java 21 LTS |
| Testing | JUnit 5 + **MockK** + SpringMockK |
| Security | Spring Security 6 (**Kotlin DSL**) |
| Async | Coroutines + Flow |
| JPA plugins | all-open + no-arg |

## Key Kotlin Rules

| Rule | Why |
|------|-----|
| `open class` for JPA entities | Hibernate needs proxies |
| `data class` for DTOs | Immutable, value semantics |
| `@field:` validation targets | Without it, constraints are silently ignored |
| MockK (not Mockito) | Mockito struggles with Kotlin `final` classes |
| `!!` is forbidden | Use `?.`, `?:`, `requireNotNull()` instead |
| Kotlin DSL for Security | `http { }` is idiomatic for Kotlin |

```
.
├── .claude
│   ├── agents
│   │   ├── code-reviewer.md
│   │   ├── devops-engineer.md
│   │   ├── docker-expert.md
│   │   ├── kotlin-architect.md        # Kotlin/Spring architect
│   │   ├── kubernetes-specialist.md
│   │   ├── security-engineer.md
│   │   ├── spring-boot-engineer.md    # Kotlin Spring engineer
│   │   └── test-automator.md
│   ├── settings.local.json
│   └── skills
│       ├── README.md
│       ├── api-contract-review
│       │   └── SKILL.md
│       ├── clean-code
│       │   └── SKILL.md
│       ├── design-patterns
│       │   └── SKILL.md
│       ├── java-architect             # Kotlin architect skill
│       │   ├── SKILL.md
│       │   └── references
│       │       ├── jpa-optimization.md
│       │       ├── reactive-webflux.md    # Coroutines + Flow
│       │       ├── spring-boot-setup.md   # Kotlin pom.xml
│       │       ├── spring-security.md     # Kotlin DSL
│       │       └── testing-patterns.md    # MockK patterns
│       ├── java-code-review           # Kotlin code review skill
│       │   └── SKILL.md
│       ├── jpa-patterns               # Kotlin JPA (open class, @field:)
│       │   └── SKILL.md
│       ├── logging-patterns
│       │   └── SKILL.md
│       ├── spring-boot-engineer       # Kotlin Spring patterns
│       │   ├── SKILL.md
│       │   └── references
│       │       ├── cloud.md
│       │       ├── data.md
│       │       ├── security.md
│       │       ├── testing.md         # MockK testing
│       │       └── web.md
│       └── spring-boot-patterns       # Kotlin Spring patterns
│           └── SKILL.md
├── CLAUDE.md
├── README.md
└── pom.xml                            # Kotlin 2.x + all-open + no-arg
```
