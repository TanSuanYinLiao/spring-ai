# spring-ai-rag 模块记忆

## 模块定位
模块化 RAG（检索增强生成）管道（`spring-ai-rag`）。遵循 Modular RAG 架构论文（arXiv:2407.21059, 2312.10997, 2410.20878），将 RAG 流程拆分为 6 个可插拔阶段，每个阶段通过函数式接口解耦。作为 BaseAdvisor 插入 ChatClient 的 Advisor 链。

## 核心数据模型

### Query — RAG 查询
```java
record Query(String text, List<Message> history, Map<String, Object> context)
```
- 不可变，Builder 模式构建
- `text` — 查询文本
- `history` — 对话历史（system + user + assistant 消息）
- `context` — 上下文数据（在管道各阶段间传递）
- `mutate()` — 创建修改副本

## 6 阶段 RAG 管道

```
用户查询
  → [1] QueryTransformer     — 查询转换（改写/压缩/翻译）
  → [2] QueryExpander        — 查询扩展（1→N 个查询变体）
  → [3] DocumentRetriever    — 文档检索（向量相似度搜索）
  → [4] DocumentJoiner       — 文档合并（多查询结果去重合并）
  → [5] DocumentPostProcessor — 文档后处理（重排/过滤/压缩）
  → [6] QueryAugmenter       — 查询增强（将检索结果注入 Prompt）
  → 返回增强后的 Prompt 给 ChatModel
```

## 阶段 1：QueryTransformer — 查询转换

### 接口
```java
interface QueryTransformer extends Function<Query, Query> {
    Query transform(Query query);
}
```
- 输入一个 Query，输出转换后的 Query
- 用途：改写、压缩、翻译用户查询

### 实现类

#### RewriteQueryTransformer — 查询改写
- 用 LLM 将用户查询改写为更适合向量搜索的形式
- 去除冗余信息，使查询更简洁精准
- 可配置目标搜索系统（默认 "vector store"）
- Prompt 模板：`{target}` + `{query}` → 改写后的查询

#### CompressionQueryTransformer — 查询压缩
- 将对话历史 + 后续查询压缩为独立查询
- 解决多轮对话中后续查询依赖上下文的问题
- 只保留 USER 和 ASSISTANT 类型的消息
- Prompt 模板：`{history}` + `{query}` → 独立查询

#### TranslationQueryTransformer — 查询翻译
- 将查询翻译为目标语言（与 embedding 模型语言匹配）
- 如果查询已是目标语言则原样返回
- Prompt 模板：`{targetLanguage}` + `{query}` → 翻译后的查询

## 阶段 2：QueryExpander — 查询扩展

### 接口
```java
interface QueryExpander extends Function<Query, List<Query>> {
    List<Query> expand(Query query);
}
```
- 输入一个 Query，输出多个 Query 变体
- 用途：从不同角度扩展查询，增加召回率

### MultiQueryExpander — 多查询扩展器
- 用 LLM 生成 N 个语义多样化的查询变体
- 默认 `numberOfQueries = 3`
- 默认 `includeOriginal = true`（结果包含原始查询）
- 如果 LLM 返回的变体数量不匹配，降级返回原始查询
- Prompt 模板：`{number}` + `{query}` → N 行查询变体

## 阶段 3：DocumentRetriever — 文档检索

### 接口
```java
interface DocumentRetriever extends Function<Query, List<Document>> {
    List<Document> retrieve(Query query);
}
```

### VectorStoreDocumentRetriever — 向量存储检索器
- 基于 VectorStore 进行语义相似度搜索
- 配置参数：
  - `vectorStore` — 必需，向量存储实例
  - `similarityThreshold` — 相似度阈值（默认接受所有）
  - `topK` — 返回 top K 结果（默认 10）
  - `filterExpression` — 元数据过滤表达式（支持 Supplier 懒加载）
- 支持通过 Query context 传入动态过滤表达式（`FILTER_EXPRESSION` key）
- 过滤表达式支持两种格式：`Filter.Expression` 对象 或 字符串（自动解析）

## 阶段 4：DocumentJoiner — 文档合并

### 接口
```java
interface DocumentJoiner extends Function<Map<Query, List<List<Document>>>, List<Document>> {
    List<Document> join(Map<Query, List<List<Document>>> documentsForQuery);
}
```
- 输入：Map<查询, List<数据源的文档列表>>
- 输出：合并后的单一文档列表

### ConcatenationDocumentJoiner — 拼接合并器
- 将所有查询、所有数据源的文档拼接在一起
- 按 document ID 去重（保留第一个出现的）
- 按 score 降序排列
- 默认实现

## 阶段 5：DocumentPostProcessor — 文档后处理

### 接口
```java
interface DocumentPostProcessor extends BiFunction<Query, List<Document>, List<Document>> {
    List<Document> process(Query query, List<Document> documents);
}
```
- 输入：原始查询 + 检索到的文档
- 输出：处理后的文档列表
- 用途：重排序、过滤、压缩

## 阶段 6：QueryAugmenter — 查询增强

### 接口
```java
interface QueryAugmenter extends BiFunction<Query, List<Document>, Query> {
    Query augment(Query query, List<Document> documents);
}
```

### ContextualQueryAugmenter — 上下文查询增强器
- 将检索到的文档内容注入到用户 Prompt 中
- 默认 Prompt 模板：
  ```
  Context information is below.
  ---------------------
  {context}
  ---------------------
  Given the context information and no prior knowledge, answer the query.
  Query: {query}
  ```
- 空上下文处理：
  - `allowEmptyContext = false`（默认）：返回 "知识库外" 的拒绝提示
  - `allowEmptyContext = true`：原样返回用户查询
- 可自定义 `documentFormatter`（默认用换行拼接文档文本）

## RetrievalAugmentationAdvisor — 管道组装器

### 核心类
- 实现 `BaseAdvisor`，可插入 ChatClient 的 Advisor 链
- 唯一必需参数：`documentRetriever`
- 其他阶段都有默认实现或可选

### Builder 配置
```java
RetrievalAugmentationAdvisor.builder()
    .queryTransformers(transformer1, transformer2)  // 可多个，链式执行
    .queryExpander(multiQueryExpander)              // 可选
    .documentRetriever(vectorStoreRetriever)         // 必需
    .documentJoiner(concatenationJoiner)             // 默认 ConcatenationDocumentJoiner
    .documentPostProcessors(processor1, processor2)  // 可多个，链式执行
    .queryAugmenter(contextualAugmenter)             // 默认 ContextualQueryAugmenter
    .taskExecutor(customExecutor)                    // 可选，默认 4 核 16 最大线程池
    .order(0)                                        // Advisor 排序
    .build();
```

### before() 执行流程（核心）
```
1. 从 ChatClientRequest 提取用户消息 → 构建 Query
2. 链式执行 QueryTransformer 列表
3. QueryExpander 扩展为多个查询
4. 并行（CompletableFuture + TaskExecutor）执行 DocumentRetriever
5. DocumentJoiner 合并结果
6. 链式执行 DocumentPostProcessor 列表
7. QueryAugmenter 将文档注入 Prompt
8. 返回增强后的 ChatClientRequest
```

### after() 执行流程
- 将检索到的文档列表（`DOCUMENT_CONTEXT`）附加到 ChatResponse 的 metadata 中
- 常量 key：`DOCUMENT_CONTEXT = "rag_document_context"`

### 默认线程池配置
```java
corePoolSize = 4
maxPoolSize = 16
threadNamePrefix = "ai-advisor-"
taskDecorator = ContextPropagatingTaskDecorator  // 传播上下文（如 MDC、SecurityContext）
```

## 使用示例

```java
// 最简用法：只需配置 DocumentRetriever
var ragAdvisor = RetrievalAugmentationAdvisor.builder()
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(pgVectorStore)
        .topK(5)
        .similarityThreshold(0.75)
        .build())
    .build();

String answer = ChatClient.create(chatModel)
    .prompt()
    .advisors(ragAdvisor)
    .user("Spring Boot 如何配置数据库？")
    .call()
    .content();

// 完整用法：所有阶段
var ragAdvisor = RetrievalAugmentationAdvisor.builder()
    .queryTransformers(
        CompressionQueryTransformer.builder().chatClientBuilder(chatClientBuilder).build(),
        RewriteQueryTransformer.builder().chatClientBuilder(chatClientBuilder).build()
    )
    .queryExpander(MultiQueryExpander.builder()
        .chatClientBuilder(chatClientBuilder)
        .numberOfQueries(3)
        .build())
    .documentRetriever(VectorStoreDocumentRetriever.builder()
        .vectorStore(vectorStore)
        .topK(5)
        .build())
    .documentPostProcessors(reranker)
    .build();
```

## 文件结构
```
spring-ai-rag/
├── Query.java                                      — 核心数据 record
├── advisor/
│   └── RetrievalAugmentationAdvisor.java           — 管道组装器（BaseAdvisor）
├── preretrieval/query/
│   ├── transformation/
│   │   ├── QueryTransformer.java                   — 接口
│   │   ├── RewriteQueryTransformer.java            — 查询改写
│   │   ├── CompressionQueryTransformer.java        — 查询压缩
│   │   └── TranslationQueryTransformer.java        — 查询翻译
│   └── expansion/
│       ├── QueryExpander.java                      — 接口
│       └── MultiQueryExpander.java                 — 多查询扩展
├── retrieval/
│   ├── search/
│   │   ├── DocumentRetriever.java                  — 接口
│   │   └── VectorStoreDocumentRetriever.java       — 向量检索实现
│   └── join/
│       ├── DocumentJoiner.java                     — 接口
│       └── ConcatenationDocumentJoiner.java        — 拼接合并实现
├── postretrieval/document/
│   └── DocumentPostProcessor.java                  — 接口
├── generation/augmentation/
│   ├── QueryAugmenter.java                         — 接口
│   └── ContextualQueryAugmenter.java               — 上下文注入实现
└── util/
    └── PromptAssert.java                           — Prompt 模板断言工具
```

## 关键设计模式
1. **函数式接口**：6 个阶段都是函数式接口（Function/BiFunction），方便组合
2. **Builder 模式**：所有实现类都用 Builder 构建
3. **不可变数据**：Query 是 record，通过 mutate() 创建修改副本
4. **管道模式**：RetrievalAugmentationAdvisor 组装各阶段为管道
5. **Advisor 模式**：作为 BaseAdvisor 插入 ChatClient 的 Advisor 链，与日志、记忆等横切关注点无缝集成
6. **并行检索**：多个查询通过 CompletableFuture + TaskExecutor 并行检索
