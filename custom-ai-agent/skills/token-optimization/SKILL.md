---
name: token-optimization
description: >
  Token-optimization guidelines and Caveman Prompting Mode for GitHub Copilot in JetBrains IntelliJ.
  Maximizes response quality while minimizing token consumption and context bloat.
---

# Token Optimization & Caveman Mode Skill

## Purpose
To achieve maximum clarity, technical accuracy, and engineering depth while spending the minimum necessary tokens. Eliminates conversational bloat, redundant code dumps, and verbose prologues.

---

## 1. The "Caveman Approach" — What it is & Token Savings

"Caveman Mode" (terse, stripped-down language) eliminates articles (*a, an, the*), pleasantries, filler phrases, and verbose transitions in both user prompts and model responses.

### Token Savings Breakdown
- **Input Tokens**: **30–50% reduction** by removing filler words and polite fluff.
- **Output Tokens**: **40–70% reduction** by receiving dense code fragments and bullet points instead of paragraphs.
- **Latency / Generation Speed**: **2x faster responses** (fewer output tokens = faster TTFT and completion time).

---

## 2. Caveman Mode Comparison Matrix

| Query Type | Standard Verbose Mode (~150 Tokens) | Caveman Mode (~35 Tokens) | Token Saved |
|---|---|---|---|
| **Spring Batch Config** | "Could you please explain how to configure Spring Batch 5 chunk size and what transaction manager to pass to StepBuilder?" | `Spring Batch 5 chunk step config? Show Java code only.` | **~75%** |
| **Virtual Threads** | "I am facing an issue where my virtual threads are getting pinned. How can I fix pinned virtual threads in Java 21?" | `Java 21 virtual thread pinning fix? Replace synchronized with ReentrantLock snippet.` | **~70%** |
| **Resilience4j** | "What is the Resilience4j annotation for circuit breaker and how do I set fallback method?" | `Resilience4j @CircuitBreaker + fallbackMethod syntax? Java code.` | **~65%** |
| **SQL N+1 Query** | "Can you check why Hibernate is executing multiple SELECT queries for my Order entity relationships?" | `Hibernate N+1 query fix? Show @EntityGraph example.` | **~70%** |

---

## 3. How to Enable Caveman Prompting in Custom Agents

You can add a directive in your `.agent.md` persona or user prompt:

```markdown
# Caveman Directive for Copilot Agents
"Mode: Caveman. Zero fluff. No greetings. No articles where unnecessary. Ultra-dense technical code & bullet points only."
```

### Example Caveman Model Response
```java
// Caveman response: Zero fluff, direct code & 1-line rule
// Rule: Pass JobRepository + PlatformTransactionManager explicitly in Spring Batch 5
@Bean
public Step orderStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("orderStep", repo)
        .<Order, ProcessedOrder>chunk(100, tx)
        .reader(reader).writer(writer)
        .build();
}
```

---

## 4. Core Principles of Token Efficiency

### A. High Signal-to-Noise Ratio (SNR)
- **Zero Filler**: Eliminate introductory chatter ("Sure, I'd be happy to help with that!"), standard politeness, and recap disclaimers.
- **Direct Entry**: Start responses directly with the answer, question, hint, or spec delta.
- **Dense Formatting**: Use tables, concise lists, and brief code fragments instead of giant blocks of standard boilerplate.

### B. Differential Snippets vs. Full File Dumps
- **Never dump whole files** unless explicitly requested or creating a new file from scratch.
- **Use Targeted Diffs / Signature Focus**:
  Show only the relevant method, class signature, or changed lines:
  ```java
  // Focus only on the modified method
  @Bean
  public Job importUserJob(JobRepository jobRepository, Step step1) {
      return new JobBuilder("importUserJob", jobRepository)
          .start(step1)
          .build();
  }
  ```

### C. Context Trimming & Selective Loading
- Guide Copilot to read only relevant files or symbols rather than loading entire repositories into context.
- When referencing specs or code, use targeted file pointers (e.g., `OrderService.java#L45-L80`).
