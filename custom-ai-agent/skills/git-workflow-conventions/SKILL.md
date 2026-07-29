---
name: git-workflow-conventions
description: >
  Git Flow, Conventional Commits, PR guidelines, and release engineering skill for 
  GitHub Copilot in IntelliJ. Ensures standardized version control and clean commit history.
---

# Git Workflow & Commit Conventions Skill

## Purpose
Establishes enterprise-grade Git standards, Conventional Commits formatting, Pull Request strategies, and release workflow conventions for Java/Spring Boot repositories.

---

## 1. Branching Strategy (Git Flow / Feature Branching)

| Branch Pattern | Purpose | Base Branch | Target PR Branch |
|---|---|---|---|
| `main` / `master` | Production-ready state | N/A | Protected |
| `develop` | Integration branch for next release | `main` | Protected |
| `feature/<jira-id>-<short-desc>` | New feature or capability | `develop` | `develop` |
| `bugfix/<jira-id>-<short-desc>` | Defect fix during sprint | `develop` | `develop` |
| `hotfix/<jira-id>-<short-desc>` | Production emergency patch | `main` | `main` & `develop` |
| `release/v<X.Y.Z>` | Release staging & validation | `develop` | `main` & `develop` |

---

## 2. Conventional Commits Specification

Format: `<type>(<scope>): <short summary in imperative mood>`

### Types
- `feat`: A new feature for the user or system (bumps MINOR version in SemVer).
- `fix`: A bug fix for the user or system (bumps PATCH version in SemVer).
- `refactor`: A code change that neither fixes a bug nor adds a feature.
- `perf`: A code change that improves performance (e.g., query optimization, virtual thread tuning).
- `test`: Adding missing tests or correcting existing tests.
- `docs`: Documentation changes only (e.g., README, API specs).
- `style`: Changes that do not affect code logic (white-space, formatting).
- `chore`: Build process, dependency updates, or tool configurations.

### Examples
```bash
# Good Feature Commit
feat(order-service): add idempotent order creation with Redis lock

# Good Bugfix Commit
fix(batch-job): resolve HikariCP connection leak during chunk retry

# Good Breaking Change Commit
feat(api-v2)!: migrate customer endpoint response to ProblemDetail RFC 7807

BREAKING CHANGE: http_status field replaced with standard status property.
```

---

## 3. Pull Request (PR) Standard Template

```markdown
## Summary of Changes
- Implemented `OrderBatchProcessor` using Spring Batch 5.x chunk processing.
- Added Resilience4j `@CircuitBreaker` wrapper around inventory RPC client.

## Linked Issues / Specs
- Spec: `.agents/specs/003-order-batch-ingestion.spec.md`
- Jira: `JAVA-1234`

## Verification & Testing
- [x] Unit tests added/updated (`JUnit 5`, `Mockito`)
- [x] Integration tests verified (`Testcontainers` with PostgreSQL)
- [x] Spring Batch step execution verified (`JobLauncherTestUtils`)
- [x] No N+1 query issues (verified via SQL log inspector)

## Security & Performance Checklist
- [x] Sensitive fields masked in logs (MDC)
- [x] Virtual thread carrier thread pinning checked (`-Djdk.tracePinnedThreads=short`)
```

---

## 4. Rebase & Clean History Guidelines
1. **Never merge `develop` into feature branch with a merge commit**: Always use `git rebase origin/develop` to keep linear history.
2. **Squash micro-commits before PR review**: Combine `WIP` or fixup commits into atomic Conventional Commits using `git rebase -i`.
3. **Commit Message Rules**:
   - Use imperative mood: "add feature" not "added feature" or "adds feature".
   - Keep subject line under 72 characters.
   - Do not end subject line with a period.
