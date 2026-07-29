# End-to-End AI & LLM Engineering Master Cheat Sheet for Backend Engineers

A production-grade, comprehensive reference guide covering **AI/LLM Core Fundamentals**, **Retrieval-Augmented Generation (RAG)**, **Vector Databases**, **AI Agent Architecture & MCP**, **Spring AI & LangChain4j Java Engineering**, and **Production LLM System Design**.

---

## 🧠 1. Core AI & LLM Fundamentals

### 1.1 The Transformer Architecture
Modern Large Language Models (LLMs) like GPT-4, Claude 3.5, and Llama 3 are built on the **Decoder-only Transformer Architecture** (Vaswani et al., 2017).

```text
[ Input Text ] ──► [ Tokenizer ] ──► [ Input Embeddings ] ──► [ Positional Encoding ]
                                                                      │
                                                                      ▼
                                                       ┌──────────────────────────────┐
                                                       │  Multi-Head Self-Attention   │
                                                       │  (Q, K, V Matrix Multiplies) │
                                                       └──────────────┬───────────────┘
                                                                      │
                                                                      ▼
                                                       ┌──────────────────────────────┐
                                                       │  Feed-Forward Network (FFN)  │
                                                       └──────────────┬───────────────┘
                                                                      │
                                                                      ▼
[ Token Probability Distribution ] ◄── [ Softmax ] ◄── [ Layer Normalization ]
```

- **Self-Attention Mechanism**: Computes contextual relationships between all words in a sequence simultaneously using three learned matrices:
  - **Query ($Q$)**: What the current token is searching for.
  - **Key ($K$)**: What information other tokens hold.
  - **Value ($V$)**: The actual representation of the token content.

$$\text{Attention}(Q, K, V) = \text{Softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

### 1.2 Key LLM Hyperparameters

| Parameter | Meaning | Production Guidance |
| :--- | :--- | :--- |
| **Temperature ($0.0 - 2.0$)** | Controls randomness of output probabilities. | Use `0.0 - 0.2` for deterministic code/JSON generation; use `0.7 - 1.0` for creative writing. |
| **Top-P (Nucleus Sampling)** | Considers only tokens comprising top $P\%$ cumulative probability mass. | Alternative to Temperature. Set Top-P to `0.9` and Temperature to `1.0`. |
| **Context Window** | Maximum combined tokens (Prompt + Response) the model can process in one call. | e.g. 128k tokens (GPT-4o), 200k tokens (Claude 3.5). Exceeding context causes truncation or `ContextWindowExceeded` error. |
| **Tokens & BPE** | Sub-word units parsed via Byte-Pair Encoding (BPE). | $\sim 1 \text{ token} \approx 0.75 \text{ words}$ (or 4 characters in English). |
| **Frequency / Presence Penalty** | Penalizes tokens based on current frequency / prior presence in output. | Prevents repetitive loops in generated output. |

---

## 🗄️ 2. Retrieval-Augmented Generation (RAG) & Vector Search

RAG solves the core limitations of LLMs: **hallucinations**, **outdated training data**, and **lack of private enterprise data context**.

### 2.1 Complete RAG Architecture Flow

```text
[ Ingestion Pipeline ]
  Documents (PDF/MD/HTML) ──► [ Chunking ] ──► [ Embedding Model ] ──► [ Vector DB (Pgvector) ]

[ Query Pipeline ]
  User Request ──► [ Query Embedding ] ──► [ Vector Search (Cosine) ] ──► Context Docs
                                                                                 │
  User Request + Retrieved Context ──► [ Guardrailed Prompt ] ──► [ LLM ] ──► Final Response
```

### 2.2 Vector Embeddings & Similarity Metrics
An **Embedding** converts raw text into a high-dimensional dense floating-point vector (e.g. 1,536 dimensions for OpenAI `text-embedding-3-small`).

- **Cosine Similarity**: Measures the cosine of the angle between two vectors (range: $-1$ to $1$). Best for text semantics regardless of length.
  $$\text{Cosine}(\vec{A}, \vec{B}) = \frac{\vec{A} \cdot \vec{B}}{\|\vec{A}\| \|\vec{B}\|}$$
- **Dot Product**: Measures magnitude and direction. Faster computation when vectors are normalized to unit length.
- **Euclidean Distance ($L2$)**: Measures straight-line distance between vector endpoints.

### 2.3 Vector Database Indexing: HNSW
Standard $O(N)$ linear vector search is too slow for millions of documents. Modern Vector DBs use **HNSW (Hierarchical Navigable Small World)** graph indexing:
- Provides $O(\log N)$ approximate nearest neighbor (ANN) search latency.
- Supported natively in **PostgreSQL via `pgvector` extension**, **Pinecone**, **Qdrant**, and **Milvus**.

```sql
-- PostgreSQL Pgvector Extension Setup
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_embeddings (
    id UUID PRIMARY KEY,
    content TEXT NOT NULL,
    metadata JSONB,
    embedding vector(1536) -- 1536-dimensional vector
);

-- Create HNSW Index for Fast Cosine Similarity Searches
CREATE INDEX idx_embeddings_hnsw ON document_embeddings 
USING hnsw (embedding vector_cosine_ops);
```

### 2.4 Chunking Strategies
- **Fixed-Size Chunking**: Chunks text into fixed token lengths (e.g. 512 tokens with 50-token overlap).
- **Recursive Character Chunking**: Recursively splits on paragraphs (`\n\n`), sentences (`.`), and words to preserve natural boundaries.
- **Semantic Chunking**: Computes embedding distances between consecutive sentences and splits when semantic drift exceeds a threshold.

---

## 🤖 3. AI Agent Engineering & Model Context Protocol (MCP)

### 3.1 The ReAct Pattern (Reasoning + Acting)
An **AI Agent** is an autonomous loop that combines an LLM with external **Tools / Functions** and **Memory**.

```text
User Goal ──► [ Thought: "I need to check user account balance." ]
                   │
                   ▼
              [ Action: Call Tool `getUserBalance(userId=42)` ]
                   │
                   ▼
              [ Observation: "Balance = $1,500.00" ]
                   │
                   ▼
              [ Thought: "Now I can answer the user query." ]
                   │
                   ▼
              [ Final Answer: "Your current balance is $1,500.00." ]
```

### 3.2 Model Context Protocol (MCP) by Anthropic
**MCP** is an open, universal standard protocol that decouples LLM applications from external tools and data sources.

```text
[ AI Agent / IDE Client (Cursor/Copilot) ]
                    │
                    ▼ (JSON-RPC over stdio / HTTP-SSE)
          [ MCP Host / Client ]
                    │
       ┌────────────┴────────────┐
       ▼                         ▼
[ Local DB MCP Server ]   [ GitHub API MCP Server ]
```

- **MCP Client**: Exposes tools, prompts, and resources to the LLM agent.
- **MCP Server**: Implements specific capabilities (e.g. executing SQL queries, querying git repos, interacting with REST APIs).

---

## ☕ 4. Spring AI & Java AI Ecosystem

Spring AI provides a unified, portable abstraction layer over AI models, vector stores, and prompt templates in Spring Boot 3.

### 4.1 Production Maven Dependencies

```xml
<dependencies>
    <!-- Spring AI OpenAI Starter -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-openai-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
    
    <!-- Spring AI Pgvector Store Starter -->
    <dependency>
        <groupId>org.springframework.ai</groupId>
        <artifactId>spring-ai-pgvector-store-spring-boot-starter</artifactId>
        <version>1.0.0-M1</version>
    </dependency>
</dependencies>
```

### 4.2 Spring Boot 3 + Spring AI Function Calling Implementation

Spring AI allows declaring standard Java `@Bean` functions that LLMs can automatically invoke.

```java
package com.example.demo.ai;

import org.springframework.ai.chat.model.ChatModel;
import org.springframework.ai.chat.prompt.Prompt;
import org.springframework.ai.openai.OpenAiChatOptions;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Description;
import org.springframework.stereotype.Service;

import java.util.function.Function;

@Configuration
public class AiToolConfig {

    public record AccountRequest(Long userId) {}
    public record AccountResponse(Long userId, double balance, String status) {}

    // Register a Tool function accessible to the LLM
    @Bean
    @Description("Fetch bank account details and current balance for a given user ID")
    public Function<AccountRequest, AccountResponse> fetchAccountDetails() {
        return request -> {
            // Real business logic execution
            return new AccountResponse(request.userId(), 2450.75, "ACTIVE");
        };
    }
}

@Service
public class AiAgentService {

    private final ChatModel chatModel;

    public AiAgentService(ChatModel chatModel) {
        this.chatModel = chatModel;
    }

    public String processUserRequestWithTools(String userPrompt) {
        // Configure LLM options to enable function calling
        OpenAiChatOptions options = OpenAiChatOptions.builder()
                .withFunction("fetchAccountDetails") // Enable registered tool
                .build();

        Prompt prompt = new Prompt(userPrompt, options);
        return chatModel.call(prompt).getResult().getOutput().getContent();
    }
}
```

---

## 🛡️ 5. Production LLM System Design & Best Practices

### 5.1 Hallucination Mitigation
1. **Strict System Prompt Constraints**: Explicitly instruct the model: *"Answer strictly using the provided context. If unknown, state 'Data unavailable'."*
2. **Low Temperature ($0.0$)**: Forces greedy decoding, reducing stochastic generation.
3. **Citation Enforcement**: Require the model to output source document IDs for every claim.

### 5.2 Semantic Caching with Redis
Caching exact string queries misses similar rephrased queries. **Semantic Caching** embeds user queries and checks Redis vector similarity:
- If query embedding distance to cached entry is $< 0.05$, return cached response instantly. Saves LLM API costs and reduces latency from $2\text{s}$ to $10\text{ms}$.

---

## 💡 Senior Interview Summary Tips

1. **Explain the Dual Pipeline of RAG**: Ingestion (Chunking $\rightarrow$ Embedding $\rightarrow$ Vector DB) vs. Query (User query $\rightarrow$ Vector search $\rightarrow$ Context Prompt $\rightarrow$ LLM).
2. **Know HNSW Indexing**: Explain how HNSW graphs enable $O(\log N)$ ANN vector search inside PostgreSQL (`pgvector`).
3. **Know MCP (Model Context Protocol)**: Highlight MCP as the modern open standard for connecting AI agents to enterprise APIs and databases.
