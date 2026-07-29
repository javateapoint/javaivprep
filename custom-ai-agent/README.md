# Custom GitHub Copilot AI Agents & Skills Suite for JetBrains IntelliJ IDEA

### Master Senior Java Backend Engineering & AI Agent Suite (Java 21 · Spring Boot 3.x · Spring Batch 5.x · DevSecOps · SRE · JVM Tuning · Git Flow)

This repository contains an end-to-end, enterprise-grade suite of **GitHub Copilot Custom Agents** (`.agent.md`), **Modular Skills** (`SKILL.md`), **Spec Lifecycle Templates**, and **Token Optimization Cheat Sheets** engineered for senior Java backend developers, architects, and engineering leads using JetBrains IntelliJ IDEA.

---

## Complete Suite File Hierarchy

```
custom-ai-agent/
├── README.md                                  # Complete Suite Master Guide (this file)
├── caveman-prompt-cheatsheet.md              # Caveman Mode Prompt Cheat Sheet (60-80% Token Savings)
├── custom-ai-agents-beginner-to-pro-guide.md  # Framework-Agnostic Agent Building Guide
├── .github/
│   └── copilot-instructions.md                # Global Workspace Instructions for Copilot
├── .agents/
│   └── specs/
│       ├── 00-index.md                        # Central Specification Index
│       └── 000-template.spec.md               # Standard Spec Template File
├── agents/                                    # GitHub Copilot Agent Personas (.agent.md)
│   ├── coding-mentor.agent.md                 # Socratic Mentor & Spec Lifecycle Agent
│   ├── java21-spring-architect.agent.md        # Architecture, Design Review & System Evaluation Agent
│   ├── batch-microservice-specialist.agent.md # Spring Batch 5.x & Microservice Performance Agent
│   ├── java-test-verifier.agent.md            # Testing, Edge-Case Review & TDD Agent
│   ├── java-code-reviewer.agent.md            # Code Review, DevSecOps & Quality Gate Inspector
│   └── performance-tuning-specialist.agent.md # JVM 21, Memory Leak & Virtual Thread Pinning Agent
└── skills/                                    # Reusable Technical Skills (SKILL.md)
    ├── spec-lifecycle/
    │   └── SKILL.md                           # Internal Specification Management Lifecycle Skill
    ├── token-optimization/
    │   └── SKILL.md                           # Low-Overhead, High-Signal & Caveman Mode Skill
    ├── java21-spring-best-practices/
    │   └── SKILL.md                           # Java 21, Spring Boot 3 & Batch 5 Technical Reference
    ├── git-workflow-conventions/
    │   └── SKILL.md                           # Git Flow, Conventional Commits & PR Standards Skill
    ├── security-observability-sre/
    │   └── SKILL.md                           # DevSecOps, Flyway, HikariCP & Kubernetes SRE Skill
    └── performance-jvm-tuning/
        └── SKILL.md                           # Generational ZGC, Virtual Thread Pinning & JFR Skill
```

---

## 1. Custom Agents Overview (6 Personas)

| Agent File | Copilot Command | Primary Persona & Focus | Recommended Use Cases |
|---|---|---|---|
| [`agents/coding-mentor.agent.md`](agents/coding-mentor.agent.md) | `@coding-mentor` | Socratic mentor for problem clarification, pseudocode, small incremental steps, testing, DevSecOps, Git Flow, and internal spec management. | Daily feature development, guided problem solving, spec creation. |
| [`agents/java21-spring-architect.agent.md`](agents/java21-spring-architect.agent.md) | `@java21-spring-architect` | Senior system architect evaluating Virtual Threads, Resilience4j, transaction boundaries, and REST API design. | Architectural design decisions, system trade-off reviews. |
| [`agents/batch-microservice-specialist.agent.md`](agents/batch-microservice-specialist.agent.md) | `@batch-microservice-specialist` | Specialist for Spring Batch 5.x Chunk/Tasklet steps, fault tolerance (skip/retry), partitioning, and memory tuning. | Building/tuning batch pipelines, high-throughput ETL data ingestion. |
| [`agents/java-test-verifier.agent.md`](agents/java-test-verifier.agent.md) | `@java-test-verifier` | QA & Test Architect guiding JUnit 5, Testcontainers, `@SpringBatchTest`, and edge-case discovery. | Writing unit/integration tests, reviewing edge cases and boundaries. |
| [`agents/java-code-reviewer.agent.md`](agents/java-code-reviewer.agent.md) | `@java-code-reviewer` | Quality Gate Inspector performing severity-rated PR/diff reviews across Java 21, Spring 3, security, N+1 queries, and Conventional Commits. | Pull Request reviews, diff inspections, release quality gates. |
| [`agents/performance-tuning-specialist.agent.md`](agents/performance-tuning-specialist.agent.md) | `@performance-tuning-specialist` | Principal Performance Engineer diagnosing Generational ZGC pause times, virtual thread carrier pinning, memory leaks, and HikariCP starvation. | High-latency RCA, thread dump analysis, heap memory profiling. |

---

## 2. Modular Skills Overview (6 Skills)

| Skill File | Scope & Capabilities | Key Operational Rules |
|---|---|---|
| [`skills/spec-lifecycle/SKILL.md`](skills/spec-lifecycle/SKILL.md) | Internal Spec Management | Executes `ensure_spec_index`, `create_spec`, `update_spec`, `reindex_specs`. Keeps specs behavioral and isolates implementation details. |
| [`skills/token-optimization/SKILL.md`](skills/token-optimization/SKILL.md) | Token Economy & Caveman Mode | Zero filler, direct entry, differential code snippets, and Caveman Mode (30–70% token savings). |
| [`skills/java21-spring-best-practices/SKILL.md`](skills/java21-spring-best-practices/SKILL.md) | Tech Stack Reference | Guidelines for Virtual Threads (Loom), Records, Pattern Matching, Spring Batch 5 Java Config, Resilience4j, and Micrometer Tracing. |
| [`skills/git-workflow-conventions/SKILL.md`](skills/git-workflow-conventions/SKILL.md) | Git Flow & Commit Discipline | Conventional Commits (`feat:`, `fix:`, `refactor:`), Git Flow branching (`feature/`, `bugfix/`), PR description templates, clean rebase strategies. |
| [`skills/security-observability-sre/SKILL.md`](skills/security-observability-sre/SKILL.md) | DevSecOps & Production SRE | Spring Security 6.x stateless JWT, OWASP API Top 10, Flyway SQL schema migrations, HikariCP tuning, N+1 query elimination, K8s readiness/liveness probes. |
| [`skills/performance-jvm-tuning/SKILL.md`](skills/performance-jvm-tuning/SKILL.md) | JVM 21 Performance & GC Tuning | Generational ZGC flags (`-XX:+UseZGC -XX:+ZGenerational`), virtual thread pinning flags (`-Djdk.tracePinnedThreads=full`), `jcmd` CLI commands. |

---

## 3. Quick Setup for JetBrains IntelliJ IDEA

Execute these commands from your Git repository root to immediately activate the full agent suite:

```powershell
# 1. Copy Agents to .github/agents/
mkdir .github\agents -ErrorAction SilentlyContinue
Copy-Item "custom-ai-agent\agents\*.agent.md" .github\agents\

# 2. Copy Skills to .github/skills/
mkdir .github\skills -ErrorAction SilentlyContinue
Copy-Item -Recurse "custom-ai-agent\skills\*" .github\skills\

# 3. Copy Global Copilot Instructions
Copy-Item "custom-ai-agent\.github\copilot-instructions.md" .github\

# 4. Initialize Spec Index Directory
mkdir .agents\specs -ErrorAction SilentlyContinue
Copy-Item "custom-ai-agent\.agents\specs\*" .agents\specs\
```

---

## 4. Token-Saving Caveman Prompting Guide

For daily development tasks, consult [`caveman-prompt-cheatsheet.md`](caveman-prompt-cheatsheet.md).

Simply append `Mode: Caveman` to your prompt:
> `@coding-mentor Spring Security 6 stateless JWT SecurityFilterChain code? Mode: Caveman.`

Yields an instant 70% token savings and 2x faster Copilot responses!
