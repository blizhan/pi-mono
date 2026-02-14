# UI Integration Patterns (TUI / Web / Slack)

## 1. 核心原则

UI 层只做三件事：

1. 输入采集
2. 流式呈现
3. 用户控制（停止、重试、选择）

不要在 UI 层实现 Agent 核心状态机。

---

## 2. 统一适配器接口

建议为不同前端统一一套 adapter：

```ts
interface AgentUiAdapter {
  onUserInput(input: string): Promise<void>;
  onStreamEvent(event: AgentEvent): void;
  onToolEvent(event: ToolEvent): void;
  onStop(reason: StopReason): void;
  onError(error: Error): void;
}
```

这样 TUI/Web/Slack 仅替换 adapter，不改 runtime。

---

## 3. TUI 模式

适用场景：开发者工作流、低延迟、键盘优先交互。

关键实现点：

- 差分渲染保证流式输出可读。
- 键位系统可配置，不把快捷键写死。
- 长输出分页与折叠，避免视觉阻塞。

---

## 4. Web 模式

适用场景：多用户产品化、可视化工具结果、持久化会话。

关键实现点：

- 组件化：聊天面板、工具卡片、会话列表分离。
- tool renderer 注册机制，支持业务工具定制展示。
- 会话、设置、密钥分 store 管理。
- 浏览器网络限制下引入 proxy 策略。

---

## 5. Slack/IM 模式

适用场景：企业内协作、事件驱动任务。

关键实现点：

- 渠道级运行状态隔离（每个 channel 独立）。
- 消息更新串行化，避免并发覆盖。
- 停止指令与运行状态反馈一致（stopping/stopped）。

---

## 6. 流式渲染统一策略

所有 UI 都建议遵循：

1. `text_start` 时创建消息容器。
2. `text_delta` 只做 append，不做重排。
3. `toolcall_*` 单独区块展示，避免混入正文。
4. `done` 冻结消息并写入会话。
5. `error` 与 `aborted` 分开视觉语义。

---

## 7. 交互一致性建议

跨端一致要优先统一：

- Stop 行为
- Retry 行为
- Tool 执行状态文案
- Usage/Cost 展示
- 会话命名与恢复入口

一致性高，用户迁移成本低。

---

## 8. 最小跨端交付方案

1. 先接 runtime + event stream。
2. 再做统一 message store。
3. 最后加端特定能力（快捷键、附件、文件上传、通知）。

这样可以避免 UI 先行导致协议反复改动。

