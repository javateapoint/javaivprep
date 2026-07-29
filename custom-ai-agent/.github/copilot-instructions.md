# Global Repository Instructions for GitHub Copilot in IntelliJ

## Tech Stack Baseline
- **Java**: Java 21 LTS (Virtual Threads, Records, Pattern Matching, Sealed Interfaces, Sequenced Collections).
- **Frameworks**: Spring Boot 3.x, Spring Batch 5.x, Spring Data JPA, Spring Kafka, Spring Security 6.x.
- **Resilience & Testing**: Resilience4j (`@CircuitBreaker`), JUnit 5, Testcontainers, `@SpringBatchTest`.
- **Database & Migration**: PostgreSQL/MySQL, Flyway (`db/migration/V<N>__...sql`), HikariCP.

## Global Response Rules
1. **Token Efficiency First**: No polite filler ("Sure! I'd be happy to help"). Provide high-density, direct technical solutions.
2. **Differential Code Snippets**: Never output complete 200-line classes unless explicitly asked. Output only modified methods, diffs, or signatures.
3. **Caveman Mode Support**: When prompt contains "Caveman" or "terse", reply using stripped-down code fragments and 1-line rules.
4. **Git Flow & Conventional Commits**: Default commit message suggestions to Conventional Commits (`feat:`, `fix:`, `refactor:`).
5. **Spec Alignment**: Check `.agents/specs/` for behavioral requirements before proposing structural refactoring.
