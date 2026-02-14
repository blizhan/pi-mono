# Agent Runtime Deep Dive (`packages/agent`)

## 1. 目标与边界

`packages/agent` 是业务无关的 Agent runtime，目标是把“模型调用 + 工具调用 + 状态推进”做成可组合内核。

边界：

- 负责：Agent 回合推进、工具执行流程、状态管理、中断控制。
- 不负责：UI 展示、渠道协议（Slack/HTTP）、部署运维。

---

## 2. 核心循环模型

可把核心循环抽象为：

1. 读取当前 `Context`。
2. 调用 `pi-ai` 流式生成 assistant message。
3. 识别 tool calls。
4. 执行工具并生成 tool results。
5. 将 assistant + tool result 写回上下文。
6. 根据 stop reason 决定继续/结束。

这个循环是后续所有“计划模式、多代理协作、自动修复”的基础。

---

## 3. Tool 事务（Transaction）设计

建议把工具执行视为一个事务，包含：

- 输入：schema 校验后的参数。
- 执行：超时、重试、取消控制。
- 输出：结构化 result + 附加元数据（耗时、错误码、trace id）。
- 回写：统一写回 `ToolResultMessage`。

优势：

- 易观测：每个工具调用有统一记录。
- 易恢复：失败可重放/跳过。
- 易扩展：可插入审计、策略引擎、权限系统。

---

## 4. 状态管理关键点

运行时状态通常至少包含：

- 当前消息序列
- 工具注册表
- 本轮 usage 与总 usage
- stop reason
- abort 信号状态

建议把状态分为两层：

1. **Persistent State**：会话可恢复所需（消息、模型、配置）。
2. **Ephemeral State**：仅本轮有效（流式临时片段、UI transient）。

这样能避免把瞬时状态污染持久层。

---

## 5. 中断与恢复

Agent 进入生产后，`abort` 不是可选项。

建议策略：

- 中断时保存已完成的 assistant chunk 与工具阶段。
- 下一次恢复时从“最近完整消息”继续。
- 对外统一暴露 `aborted` stop reason，避免和异常混淆。

---

## 6. 可扩展点设计

Runtime 最值得预留的扩展点：

1. `beforeModelCall` / `afterModelCall`
2. `beforeToolCall` / `afterToolCall`
3. `onStop`
4. `onError`

上层可在这些钩子里接：

- 指标采集
- 成本控制
- 审计与策略
- 安全过滤

---

## 7. 推荐的最小 runtime 接口草案

```ts
interface AgentRuntime {
  run(context: Context, opts?: RunOptions): Promise<RunResult>;
  abort(): void;
  setTools(tools: AgentTool[]): void;
  getState(): AgentState;
}
```

其中 `RunResult` 建议包含：

- `messages`（本次新增）
- `usage`（本次与累计）
- `stopReason`
- `errors`（可选）

---

## 8. 对你后续实现的建议

1. 优先保证工具事务一致性，再追求 prompt 技巧。
2. 把 runtime 做成纯逻辑层，适配器层只做输入输出转换。
3. 先落可观测性（trace、usage、stop reason）再做复杂策略。

