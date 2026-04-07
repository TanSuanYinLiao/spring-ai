# spring-ai-commons 模块记忆

## 模块定位
Spring AI 共享基础模块（`spring-ai-commons`），版本 2.0.0-SNAPSHOT，目标 Spring Boot 4.x。定义整个框架共用的核心数据结构、接口和工具类。所有其他模块（models、vector-stores、RAG、document-readers）都依赖此模块。

## 核心数据模型

### Content 层级
- `Content` — 顶层接口：`getText()` + `getMetadata()`。是 `Document` 和聊天 `Message` 的共同父接口
- `MediaContent` — 扩展 `Content`，增加 `getMedia()` 支持多模态
- `Media` — 媒体附件（MIME 类型 + URI/byte[] + id + name），内置 `Format` 常量类

### Document（核心数据类）
- 承载文本或媒体内容 + 元数据 + 评分 + 唯一 ID
- 位于 `org.springframework.ai.document.Document`
- Builder 模式 + Jackson 序列化
- `MetadataMode` 枚举：ALL / EMBED / INFERENCE / NONE（控制格式化时包含哪些元数据）

## ETL 管道接口
- `DocumentReader` extends `Supplier<List<Document>>` — 读取端
- `DocumentTransformer` extends `Function<List<Document>, List<Document>>` — 转换端
- `DocumentWriter` extends `Consumer<List<Document>>` — 写入端

## ID 生成策略（`document/id`）
- `IdGenerator` 接口
- `RandomIdGenerator` — UUID.randomUUID()（默认）
- `JdkSha256HexIdGenerator` — SHA-256 内容哈希（去重场景）

## Reader（`reader` 包）
- `TextReader` — 从 Spring Resource 读纯文本
- `JsonReader` — 从 JSON 读取，支持 key 过滤和 JSON Pointer
- `ExtractedTextFormatter` — 文本提取格式化（左对齐、删首尾行、合并空行）

## Transformer（`transformer` 包）
- `ContentFormatTransformer` — 批量更新文档的 ContentFormatter
- `TextSplitter` — 抽象文本分块基类
- `TokenTextSplitter` — 基于 token 数量的智能分块器（使用 JTokkit）

## Writer（`writer` 包）
- `FileDocumentWriter` — 将文档列表写入文件

## 评估（`evaluation` 包）
- `Evaluator` — 函数式接口：`evaluate(EvaluationRequest) → EvaluationResponse`
- 用于 RAG 回答质量评估

## 可观测性（`observation` 包）
- 遵循 OpenTelemetry GenAI 语义约定
- `AiObservationAttributes` — 标准化属性 key
- `AiOperationType` — chat/embedding/image/text_completion/framework
- `AiProvider` — openai/anthropic/ollama 等
- `VectorStoreObservationAttributes` / `VectorStoreProvider` — 向量存储观测

## 模板（`template` 包）
- `TemplateRenderer` — BiFunction<template, variables, result>
- `NoOpTemplateRenderer` — 透传不做替换

## Token 计数（`tokenizer` 包）
- `TokenCountEstimator` 接口
- `JTokkitTokenCountEstimator` — 基于 JTokkit，CL100K_BASE 编码

## 工具类（`util` 包）
- `JacksonUtils` — Jackson 模块加载
- `ParsingUtils` — 驼峰命名拆分
- `ResourceUtils` — URI 资源文本加载

## 核心依赖
- Spring Framework (spring-context)
- Micrometer (micrometer-core, context-propagation, micrometer-tracing)
- Jackson (jackson-databind)
- JTokkit (tokenization)
- Kotlin stdlib/reflect (optional)
