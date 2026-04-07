# spring-ai-client-chat 模块记忆

## 模块定位
ChatClient 流式 API + Advisor 链（`spring-ai-client-chat`）。用户与 AI 模型交互的统一入口，通过 Advisor 链实现横切关注点（日志、记忆、RAG、工具调用）。依赖 `spring-ai-model`。

## ChatClient 门面接口

### 静态工厂方法（入口）
- `create(ChatModel)` — 最简单入口
- `create(ChatModel, ObservationRegistry)` — 带观测
- `builder(ChatModel)` — Builder 模式入口，配置默认值后 `build()`
- 内部委托给 `DefaultChatClientBuilder`

### 实例方法（3 个）
- `prompt()` — 空请求，返回 ChatClientRequestSpec
- `prompt(String)` — 文本请求
- `prompt(Prompt)` — 完整 Prompt
- `mutate()` — 复制当前配置创建新 Builder

### 嵌套接口族
```
ChatClient
├── Builder              ← 创建 ChatClient，配置默认 Advisors/Options/System/User/Tools
├── ChatClientRequestSpec ← 构建请求（system/user/advisors/tools/call/stream）
├── PromptUserSpec       ← user 消息构建（text/media/params）
├── PromptSystemSpec     ← system 消息构建（text/params）
├── AdvisorSpec          ← advisor 配置
├── CallResponseSpec     ← 同步调用结果（entity/content/chatClientResponse）
└── StreamResponseSpec   ← 流式调用结果（Flux<ChatClientResponse>/Flux<String>）
```

### Builder 接口详解
- `defaultAdvisors(...)` — 配置默认 Advisors（所有请求继承）
- `defaultOptions(ChatOptions)` — 配置默认选项（温度、maxTokens 等）
- `defaultSystem(...)` / `defaultUser(...)` — 配置默认消息
- `defaultToolNames(...)` / `defaultTools(...)` / `defaultToolCallbacks(...)` — 配置默认工具
- `defaultTemplateRenderer(...)` — 配置默认模板渲染器
- `clone()` — 克隆当前配置
- `build()` — 构建 ChatClient 实例

### ChatClientRequestSpec 详解（核心）
```java
interface ChatClientRequestSpec {
    // Advisor 配置
    advisors(Advisor... advisors);
    advisors(Consumer<AdvisorSpec> consumer);

    // 消息配置
    system(String text);                    // system 消息
    system(Resource text);                  // 从文件读
    system(Consumer<PromptSystemSpec>);     // lambda 构建
    user(String text);                      // user 消息
    user(Resource text);
    user(Consumer<PromptUserSpec>);         // lambda 构建
    messages(Message... messages);          // 直接传 Message 列表

    // 工具配置
    toolNames(String... toolNames);
    tools(Object... toolObjects);
    toolCallbacks(ToolCallback... toolCallbacks);
    toolCallbacks(ToolCallbackProvider... providers);
    toolContext(Map<String, Object> toolContext);

    // 选项和模板
    options(T extends ChatOptions);
    templateRenderer(TemplateRenderer templateRenderer);

    // 执行
    call();     // 同步调用 → CallResponseSpec
    stream();   // 流式调用 → StreamResponseSpec

    mutate();   // 复制配置创建新 Builder
}
```

### PromptUserSpec 详解
- `text(String)` — 用户消息文本
- `text(Resource)` — 从文件读取消息
- `params(Map)` / `param(k, v)` — 模板参数（如 `{name}`）
- `media(Media...)` — 多模态附件（图片、音频等）
- `media(MimeType, URL)` / `media(MimeType, Resource)` — URL/Resource 形式的媒体
- `metadata(...)` — 元数据

### PromptSystemSpec 详解
- 与 PromptUserSpec 类似，但不支持 media（system 消息通常不含附件）

### AdvisorSpec 详解
- `param(k, v)` / `params(Map)` — 向 Advisor 传递参数（如 RAG 的 topK）
- `advisors(Advisor...)` — 添加 Advisor

### CallResponseSpec 详解
- `content()` — 最简单，直接返回 String 文本
- `entity(Class<T>)` — 结构化输出：JSON → Bean
- `entity(ParameterizedTypeReference<T>)` — 泛型结构化输出
- `entity(StructuredOutputConverter<T>)` — 自定义转换器
- `chatResponse()` — 获取完整 ChatResponse（含 metadata、token 用量）
- `chatClientResponse()` — 获取 ChatClientResponse（含 context）
- `responseEntity(Class<T>)` — 返回 ResponseEntity（响应 + 实体）

### StreamResponseSpec 详解
- `content()` — Flux<String> 纯文本流
- `chatResponse()` — Flux<ChatResponse> 响应流
- `chatClientResponse()` — Flux<ChatClientResponse> 完整响应流

### 典型用法
```java
// 基本用法
ChatClient.create(chatModel)
    .prompt()
    .system("你是一个助手")
    .user("你好")
    .advisors(new SimpleLoggerAdvisor())
    .call()
    .content();

// 模板化 user 消息
.user(u -> u.text("翻译: {text}").param("text", "Hello"))

// 多模态
.user(u -> u.text("描述这张图片").media(Media.Format.IMAGE_PNG, imageUrl))

// 结构化输出
.call().entity(Person.class);

// 流式输出
.stream().content().subscribe(token -> System.out.print(token));

// Builder 配置默认值
ChatClient.builder(chatModel)
    .defaultSystem("你是一个助手")
    .defaultAdvisors(new SimpleLoggerAdvisor())
    .build();
```

## Request/Response 数据模型

- `ChatClientRequest` — record：`Prompt + Map<String, Object> context`，不可变，通过 `mutate()` 创建修改副本
- `ChatClientResponse` — record：`ChatResponse + Map<String, Object> context`，同样不可变
- context 在 Advisor 链中传递，用于跨 Advisor 共享数据（如 RAG 文档上下文）

## Advisor 接口族

### 继承关系
```
Advisor (extends Ordered)
├── CallAdvisor      ← adviseCall(Request, Chain) → Response
├── StreamAdvisor    ← adviseStream(Request, Chain) → Flux<Response>
└── BaseAdvisor      ← extends both，提供 before()/after() 模板方法
```

### 核心接口
- `Advisor` — 根接口：`getName()` + `DEFAULT_CHAT_MEMORY_PRECEDENCE_ORDER`
- `CallAdvisor` — 同步调用 Advisor
- `StreamAdvisor` — 流式调用 Advisor
- `BaseAdvisor` — 同时实现两者，提供 `before()`/`after()` 模板 + 默认 scheduler
- `AdvisorChain` — 链根接口，持有 `ObservationRegistry`
- `CallAdvisorChain` — 同步链：`nextCall(Request) → Response`
- `StreamAdvisorChain` — 流式链：`nextStream(Request) → Flux<Response>`

### BaseAdvisor 核心逻辑
```
adviseCall:  before(request) → chain.nextCall(request) → after(response)
adviseStream: before(request) → chain.nextStream(request) → after(response) on each
```

## 内置 Advisor 实现

| Advisor | 用途 |
|---------|------|
| `SimpleLoggerAdvisor` | 日志记录请求/响应（最简单的例子） |
| `ChatModelCallAdvisor` | 链的终点：实际调用 ChatModel.call() |
| `ChatModelStreamAdvisor` | 链的终点：实际调用 ChatModel.stream() |
| `MessageChatMemoryAdvisor` | 聊天记忆：管理会话历史 |
| `PromptChatMemoryAdvisor` | 聊天记忆：另一种实现 |
| `SafeGuardAdvisor` | 安全防护 |
| `ToolCallAdvisor` | 工具调用管理 |
| `StructuredOutputValidationAdvisor` | 结构化输出验证 |
| `LastMaxTokenSizeContentPurger` | Token 数量限制，清理旧消息 |

## 链执行流程
```
ChatClient.prompt().user("hello").call()
    → DefaultChatClient 构建 ChatClientRequest
    → DefaultAroundAdvisorChain 按 order 排序
    → Advisor.before() 链式调用
    → ChatModelCallAdvisor 调用 chatModel.call()
    → Advisor.after() 反向链式调用
    → CallResponseSpec.content() 返回结果
```

## 核心实现类
- `DefaultChatClient` — ChatClient 接口的实现（最大的文件）
- `DefaultChatClientBuilder` — Builder 实现
- `DefaultAroundAdvisorChain` — Advisor 链核心实现，维护 Advisor 列表，链式调用
- `ChatClientMessageAggregator` — 流式响应聚合（Flux → Mono）

## 评估器
- `RelevancyEvaluator` — 相关性评估（基于 LLM 判断回答是否相关）
- `FactCheckingEvaluator` — 事实检查评估

## 可观测性
- `observation/*` — Micrometer 观测上下文和约定（ChatClient 级别）
- `advisor/observation/*` — Advisor 级别观测

## 关键依赖
- spring-ai-model（ChatModel, Prompt, ChatResponse, Message）
- spring-ai-commons（Media）
- spring-ai-rag（RetrievalAugmentationAdvisor 作为 Advisor 实现）
- Micrometer（观测）
- Reactor（Flux 流式处理）
