### 1. Plan Mode Default
- Enter plan mode for ANY not-trivial task (3+ steps or architectural decisions)
- Use plan mode for verification steps, not just building
- Write detailed specs upfront to reduce ambiguity

### 2. Self-Improvement Loop
- After ANY correction from the user: update `tasks/lessons.md` with the pattern
- Write rules for yourself that prevent the same mistake
- Ruthlessly iterate on these lessons until the mistake rate drops
- Review lessons at session start for a project

### 3. Verification Before Done
- Never mark a task complete without proving it works
- Diff behavior between main and your changes when relevant
- Ask yourself: "Would a staff engineer approve this?"
- Run tests, check logs, demonstrate correctness

### 4. Demand Elegance (Balanced)
- For non-trivial changes: pause and ask "is there a more elegant way?"
- If a fix feels hacky: "Knowing everything I know now, implement the elegant solution"
- Skip this for simple, obvious fixes. Don't overengineer
- Challenge your own work before presenting it

### 5. Skills usage
- Use skills for any task that requires a capability
- Load skills from `.claude/skills/`
- Invoke skills with natural language
- Each skill is one independent capability

### 6. Subagents usage
- Use subagents liberally to keep the main context window clean
- Load subagents from `.claude/agents/`
- For complex problems, throw more compute at it via subagents
- One task per subagent for focused execution on a given tech stack

## Core Principles
- **Simplicity First**: Make every change as simple as possible. Impact minimal code
- **No Laziness**: Find root causes. No temporary fixes. Senior developer standards

## Project General Instructions

- Always use the latest versions of dependencies.
- Always write Kotlin code as the Spring Boot application.
- Always use Maven for dependency management.
- Always create test cases for the generated code both positive and negative.
- Always generate the CircleCI pipeline in the .circleci directory to verify the code.
- Minimize the amount of code generated.
- The Maven artifact name must be the same as the parent directory name.
- Use semantic versioning for the Maven project. Each time you generate a new version, bump the PATCH section of the version number.
- Use `pl.piomin.services` as the group ID for the Maven project and base Kotlin package.
- Do not use the Lombok library (use Kotlin data class instead).
- Generate the Docker Compose file to run all components used by the application.
- Update README.md each time you generate a new version.

## Kotlin-Specific Instructions

- Use `data class` for DTOs and value objects (replaces Java records/Lombok @Data).
- Use `open class` for JPA entities — required for Hibernate proxies. The `all-open` Maven plugin handles `@Entity`, `@Service`, `@Component` etc. automatically.
- Use `no-arg` Maven plugin for JPA entities — generates a zero-arg constructor for `@Entity`, `@Embeddable`, `@MappedSuperclass` automatically.
- Always annotate validation constraints with `@field:` target (e.g. `@field:NotBlank`) on constructor parameters, otherwise validation is silently ignored.
- **Never use `!!` (not-null assertion)**. Use safe-call `?.`, Elvis `?:`, `requireNotNull()`, or `checkNotNull()` instead.
- Use MockK (not Mockito) for all mocking in tests. Use `@MockkBean` (SpringMockK) in `@WebMvcTest` / `@SpringBootTest`.
- Use Kotlin DSL for Spring Security `SecurityFilterChain` configuration.
- Prefer `suspend fun` + coroutines for async operations over reactive `Mono`/`Flux` where possible.
- Use constructor injection (primary constructor) — no `@Autowired` needed in Kotlin.
- Use `companion object` for factory methods and constants inside classes.
- Prefer extension functions for utility methods that operate on a type.
- Avoid `!!` — if you find yourself writing `!!`, redesign the null handling.
- Source directory is `src/main/kotlin`, test directory is `src/test/kotlin`.
