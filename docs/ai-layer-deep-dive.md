# AI Layer Deep Dive (`packages/ai`)

## 1. 目标与边界

`packages/ai` 的核心目标是把不同模型厂商、不同 API 协议和不同能力差异，压平到统一的调用与事件接口。

边界定义：

- 负责：模型定义、Provider 能力映射、统一流式事件、上下文/工具消息结构。
- 不负责：业务工作流编排、会话产品逻辑、UI 呈现。

---

## 2. 核心抽象

### 2.1 `Api` vs `Provider`

这是本层最关键的建模决策之一。

- `Provider` 表示厂商/服务方（如 `openai`、`anthropic`、`google`）。
- `Api` 表示具体调用协议（如 `openai-responses`、`anthropic-messages`、`google-vertex`）。

好处：

1. 同一 provider 可挂多个 api 形态（例如原生 API 与兼容 API）。
2. 模型路由策略可以基于协议能力，而不仅是品牌名。
3. 兼容第三方 OpenAI-like 端点时不会污染核心 provider 语义。

### 2.2 统一消息模型

`Context`/`UserMessage`/`AssistantMessage`/`ToolResultMessage` 是跨包通用协议。

统一消息模型的价值：

- 让 `agent` 层不感知 provider 差异。
- 让 `web-ui`/`tui` 按同一结构渲染。
- 让会话持久化具备跨模型迁移能力（handoff）。

### 2.3 统一事件流语义

`stream()` 输出统一事件（如 text delta、thinking、toolcall、done、error）。

这使得：

- 前端可以写一次流式渲染逻辑。
- 上层 Agent 可以统一统计 usage/stop reason。
- Provider 适配层只需要负责“厂商事件 -> 统一事件”的转换。

---

## 3. `stream.ts` 设计要点

`stream.ts` 可以理解为“协议路由器 + 选项映射器”。

主要职责：

1. 根据模型 `api` 分发到对应 `stream<Provider>()` 实现。
2. 在 `SimpleStreamOptions` 与 provider-specific options 之间映射。
3. 做环境密钥发现（`getEnvApiKey()`）与默认行为注入。
4. 保证输出事件语义一致。

设计建议（可迁移）：

- `streamFunctions` 使用显式 map，而不是隐式命名约定，便于可控扩展。
- provider-specific 映射保持“薄层转换”，避免把业务语义放入 ai 层。
- 在映射阶段做参数兜底（如 reasoning、cacheRetention），不要在 UI 层补丁式处理。

---

## 4. Provider 接入事务模型（建议模板）

新增 provider 时，建议遵循固定事务：

1. **类型扩展**：`types.ts` 增加 `Api`、`KnownProvider`、options map。
2. **provider 实现**：新建 `providers/<provider>.ts`，只做协议转换。
3. **stream 集成**：接入 `stream.ts` 路由、option 映射、密钥发现。
4. **模型生成**：在 `scripts/generate-models.ts` 加模型发现逻辑。
5. **测试矩阵覆盖**：覆盖 stream/abort/tokens/image/tool/handoff 等。

这样可以保证：

- 功能一致性
- 类型完整性
- 兼容性回归可控

---

## 5. Option 设计策略

`StreamOptions` + `SimpleStreamOptions` 的分层值得复用：

- 通用层：`temperature`/`maxTokens`/`signal`/`headers` 等。
- 高层抽象：`reasoning`/`thinkingBudgets` 等能力导向参数。
- Provider 扩展层：在具体 API options 中补充私有字段。

建议：

- 上层应用尽量只用 `SimpleStreamOptions`。
- 仅在确有需求时下沉到 provider-specific options。
- 保持“可退化”：某 provider 不支持某能力时优雅忽略或降级。

---

## 6. 错误与停止语义

统一 `StopReason`（`stop`/`length`/`toolUse`/`error`/`aborted`）的意义：

- 可观测性一致：上层可以直接按 stop reason 做统计与告警。
- 可恢复性更好：`aborted` 与 `error` 可分开处理。
- 回放与审计简单：日志结构统一。

建议在业务层保持如下处理规则：

1. `toolUse`：进入工具执行分支。
2. `length`：触发上下文压缩或继续生成策略。
3. `aborted`：保留中间状态并允许恢复。
4. `error`：记录 provider/raw payload 信息后上报。

---

## 7. 对你构建 Agent 应用的直接落地建议

1. 先固定“统一事件协议”，再接 provider。
2. 先做单 provider 全链路稳定，再做多 provider fallback/handoff。
3. 把“模型能力发现”与“业务策略选择”分开。
4. usage/cost 从第一天就结构化落库。

