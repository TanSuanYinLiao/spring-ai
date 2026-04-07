# spring-ai-retry 模块记忆

## 模块定位
AI 调用重试机制（`spring-ai-retry`）。处理 AI API 调用时的瞬态故障（限流、服务端错误、网络超时），区分可重试和不可重试的错误。所有 Provider 实现（OpenAI、Anthropic 等）都依赖此模块。

## 异常体系

### TransientAiException — 瞬态异常（可重试）
- 继承 `RuntimeException`
- 表示"之前失败的操作重试可能成功"
- 5xx 服务端错误会抛此异常
- RetryTemplate 只重试此异常

### NonTransientAiException — 非瞬态异常（不可重试）
- 继承 `RuntimeException`
- 表示"重试也不会成功，必须修复根本原因"
- 4xx 客户端错误会抛此异常（401 认证失败、400 请求错误等）
- RetryTemplate 遇到此异常会立即放弃

## RetryUtils — 重试配置工具类

### 默认重试策略
```java
maxRetries = 10                    // 最多重试 10 次
initialInterval = 2000ms           // 初始等待 2 秒
multiplier = 5                     // 每次等待时间 ×5
maxInterval = 3 分钟               // 最长等待 3 分钟
只重试: TransientAiException + ResourceAccessException（网络错误）
```

### 两个 RetryTemplate
- `DEFAULT_RETRY_TEMPLATE` — 生产环境使用，指数退避
- `SHORT_RETRY_TEMPLATE` — 测试环境使用，100ms 初始间隔

### 错误分类逻辑（DEFAULT_RESPONSE_ERROR_HANDLER）
```
HTTP 响应错误
├── 4xx 客户端错误 → NonTransientAiException（不重试）
└── 5xx 服务端错误 → TransientAiException（重试）
```

### 执行方法
```java
RetryUtils.execute(retryTemplate, () -> {
    return chatModel.call(prompt);
});
```

### 重试时间线示例
```
第 1 次失败 → 等 2 秒
第 2 次失败 → 等 10 秒
第 3 次失败 → 等 50 秒
第 4 次失败 → 等 250 秒
第 5 次失败 → 等 180 秒（达到上限）
...最多 10 次
```

## 使用场景
- OpenAI/Anthropic 等 Provider 调用 API 时的重试
- 处理 429 限流、5xx 服务端错误
- 网络超时自动重试
- 用户可通过配置自定义重试策略
