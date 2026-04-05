# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development Commands

```shell
# Build with unit tests (requires JDK 21+, targets Java 17)
./mvnw clean package

# Build without tests
./mvnw clean install -DskipTests -Dmaven.javadoc.skip=true

# Run integration tests (requires API keys as env vars)
./mvnw clean verify -Pintegration-tests

# Run integration tests for a specific module
./mvnw verify -Pintegration-tests -pl models/spring-ai-openai

# Run a specific integration test (with retry on failure)
./mvnw -pl vector-stores/spring-ai-pgvector-store -am -Pintegration-tests \
  -Dfailsafe.failIfNoSpecifiedTests=false \
  -Dfailsafe.rerunFailingTestsCount=2 \
  -Dit.test=PgVectorStoreIT verify

# Fast integration test pass (OpenAI + PGVector + Chroma only, used in CI)
./mvnw clean verify -Pci-fast-integration-tests

# Build documentation
./mvnw -pl spring-ai-docs antora

# Format check (CI enforces formatting)
./mvnw process-sources -P checkstyle-check

# Apply code formatting
# Formatting is auto-applied via spring-javaformat-maven-plugin during process-sources

# Update license headers
./mvnw license:update-file-header -Plicense

# Check javadocs
./mvnw javadoc:javadoc
```

## Project Architecture

Spring AI is a multi-module Maven project providing Spring-friendly abstractions for AI model integration. Version 2.0.0-SNAPSHOT targeting Spring Boot 4.x.

### Core Modules

| Module | Purpose |
|---|---|
| `spring-ai-model` | Core interfaces: `Model`, `ChatModel`, `EmbeddingModel`, `ImageModel`, `TranscriptionModel`, `ModerationModel` |
| `spring-ai-client-chat` | `ChatClient` fluent API with advisor chain support |
| `spring-ai-vector-store` | `VectorStore` interface, `SearchRequest`, filter expressions |
| `spring-ai-rag` | Modular RAG pipeline: query transformation, retrieval, post-processing, augmentation |
| `spring-ai-commons` | Shared types: `Content`, `Document`, `Media`, `IdGenerator` |
| `spring-ai-test` | Reusable test base classes (`BaseVectorStoreTests`, `AbstractChatOptionsTests`) |

### Provider Modules

- `models/` — AI provider implementations (OpenAI, Anthropic, Ollama, Google GenAI, Bedrock, Mistral, DeepSeek, etc.)
- `vector-stores/` — Vector store implementations (PgVector, Milvus, Pinecone, Elasticsearch, Redis, etc.)
- `document-readers/` — PDF, Tika, Markdown, Jsoup document readers
- `memory/` — Chat memory repository implementations (Cassandra, JDBC, MongoDB, Neo4j, Redis)
- `mcp/` — Model Context Protocol client/server with annotation support

### Spring Boot Integration (3-layer pattern)

Each integration follows this pattern:

1. **Core library** (`models/spring-ai-openai`) — pure Java, no Spring Boot dependency
2. **Auto-configuration** (`auto-configurations/models/spring-ai-autoconfigure-model-openai`) — `@AutoConfiguration` with conditional beans, `@ConfigurationProperties` for settings
3. **Starter** (`spring-ai-spring-boot-starters/spring-ai-starter-model-openai`) — thin pom.xml-only module bundling auto-config + core library

Auto-config classes use `@ConditionalOnClass`, `@ConditionalOnProperty`, `@ConditionalOnMissingBean` and are registered via `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.

### Key Interfaces

- `Model<TReq, TRes>` — root abstraction with `call(TReq)` method
- `StreamingModel<TReq, TRes>` — adds `Flux<TRes> stream(TReq)`
- `ChatModel` extends `Model<Prompt, ChatResponse>` + `StreamingChatModel`
- `VectorStore` extends `DocumentWriter` + `VectorStoreRetriever`
- `Advisor` — ordered interceptor with `before()`/`after()` hooks for cross-cutting concerns
- `ToolCallback` — function calling interface with `ToolDefinition` + JSON schema
- `StructuredOutputConverter<T>` — maps AI output to POJOs

### RAG Pipeline (spring-ai-rag)

`RetrievalAugmentationAdvisor` composes modular stages:
1. `QueryTransformer` — rewrite/transform user query
2. `QueryExpander` — generate multiple query variants
3. `DocumentRetriever` — retrieve from VectorStore
4. `DocumentJoiner` — merge multi-query results
5. `DocumentPostProcessor` — rerank/filter
6. `QueryAugmenter` — inject context into prompt

## Code Style & Conventions

- **Formatter**: Spring Java Format (enforced by CI, auto-applied during build)
- **Checkstyle**: Follows Spring Framework guidelines — tabs, LF line endings
- **Javadoc**: Wrap at 90 characters; code wrap at 120 characters
- **License**: Apache 2.0 header required on all Java files (`Copyright 2023-present`)
- **Annotations**: `@since` on new public API, `@author` on all public classes
- **Commits**: DCO sign-off required (`Signed-off-by` trailer); branch naming uses GitHub issue ID (e.g., `GH-123`)
- **Java version**: Compile target is Java 17, build requires JDK 21+

## Testing Conventions

- Unit tests: `*Tests.java` (e.g., `ChatResponseTests`)
- Integration tests: `*IT.java` (e.g., `OpenAiChatModelIT`) — require API keys, skipped if unset
- Framework: JUnit 5, AssertJ, Mockito, Awaitility, Reactor Test
- Method naming: `whenConditionThenResult` pattern
- `BaseVectorStoreTests` provides template method for vector store tests
- `TestConfiguration` classes for test-specific Spring beans

## Cloning Note

This repo contains large ONNX model files. Either install Git LFS before cloning, or use `GIT_LFS_SKIP_SMUDGE=1 git clone ...` to skip them.
