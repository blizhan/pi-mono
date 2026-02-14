# pi-mono 深度研究文档（面向 Agent 应用构建）

## 1. 项目定位与整体价值

`pi-mono` 是一个围绕“可落地 Agent 系统”构建的 TypeScript Monorepo，而不是单一聊天 SDK。它把 **模型调用、Agent 运行时、交互终端、Web 组件、部署工具、Slack 集成** 拆成独立包，形成从“底层 LLM 抽象”到“业务集成入口”的完整链路。

核心价值：

- 在多模型/多供应商环境下保持统一 API 与消息格式，降低 provider 切换成本。
- 将 Agent 核心循环（工具调用、消息状态、中断恢复）独立为可复用 runtime。
- 通过 CLI/TUI/Web/Slack 多交互面复用同一 Agent 能力，减少重复开发。
- 通过 pods 包连接本地开发与 GPU 推理部署，支持从实验到生产的迁移。

---

## 2. Monorepo 架构分层（建议理解模型）

可以把项目分成 5 层：

1. **LLM 抽象层**：`packages/ai`
2. **Agent Runtime 层**：`packages/agent`
3. **交互与编排层**：`packages/coding-agent`
4. **界面层**：`packages/tui` + `packages/web-ui`
5. **集成与部署层**：`packages/mom` + `packages/pods`

建议你以后做 Agent 应用时，也采用类似“层间依赖单向流动”的设计：

- 上层依赖下层。
- 下层不感知上层 UI/业务语义。
- 通过类型稳定的消息协议（Context / Message / ToolCall / ToolResult）完成层间通信。

---

## 3. 各包职责与关键代码设计点

### 3.1 `packages/ai`（统一模型抽象）

**定位**：统一多供应商模型调用，抽象差异化 API。

关键设计点：

- **API 与 Provider 分离**：`Api` 与 `Provider` 分开建模，避免把模型厂商与协议绑定死。
- **统一流式事件语义**：对不同厂商输出对齐到统一事件流（text / thinking / toolcall / done / error）。
- **上下文可迁移**：对话上下文（含工具、工具结果）可序列化、跨模型 handoff。
- **`SimpleStreamOptions` 封装 reasoning**：在统一入口中暴露推理强度参数，映射到不同 provider 能力。
- **可扩展 Provider 机制**：新增供应商时，通过类型、stream 映射、模型生成脚本与测试矩阵完成接入。

你如果准备做“模型编排中间层”，这里的模式可直接借鉴：

- 协议归一化（事件、错误、停止原因）
- 能力探测（是否支持 tool calling、thinking、图片）
- provider-specific option 到 unified option 的映射

### 3.2 `packages/agent`（Agent 核心循环）

**定位**：负责“一个 Agent 回合”如何推进，而不是 UI。

关键设计点：

- **运行循环与 transport 解耦**：Agent 内核聚焦状态转换（消息 -> 模型调用 -> 工具调用 -> 新消息）。
- **Tool Schema 驱动**：工具参数通过 schema 定义并在调用前后验证，降低 hallucinated 参数风险。
- **可中断性**：支持 Abort / stop，便于在 CLI、Web、Slack 场景统一控制。
- **状态对象标准化**：把消息、工具结果、usage、stop reason 作为一等公民输出。

这层本质是你的“应用大脑”。如果后续你做垂直 Agent，优先复用这层，少在 UI 里写业务状态机。

### 3.3 `packages/coding-agent`（产品化 Agent 编排）

**定位**：把通用 Agent 能力产品化为可交互 CLI（含会话、扩展、模型解析、计划模式等）。

关键设计点：

- **Session 持久化与分支化**：适合长周期任务，支持上下文管理与压缩（compaction）。
- **Model Registry + Resolver**：将“默认模型/环境变量/用户配置”抽象为可解析策略。
- **Extension 系统**：把工具、流程钩子、能力开关外插，避免核心代码膨胀。
- **Prompt 模板与运行策略分离**：便于针对不同任务场景（修复、重构、分析）切换策略。

这部分非常适合你研究“如何把 Agent 从 demo 做成产品”的工程化细节。

### 3.4 `packages/tui`（终端 UI 库）

**定位**：高性能终端渲染与交互组件，不绑定特定 Agent 逻辑。

关键设计点：

- **差分渲染**：减少终端重绘，提升流式输出场景的体验。
- **输入与编辑器组件分层**：键盘事件、编辑行为、展示组件相对独立。
- **Keybinding 可配置**：将快捷键从逻辑中抽离，便于跨平台一致性。

如果你未来要做“IDE 风格 Agent CLI”，这层的可重用性很高。

### 3.5 `packages/web-ui`（Web 组件化界面）

**定位**：提供可嵌入式 Agent Web 组件（ChatPanel、工具渲染、存储、对话框）。

关键设计点：

- **UI 与 Agent runtime 解耦**：组件依赖 `Agent` 接口而非业务实现。
- **Tool Renderer 注册机制**：通过 renderer registry 扩展工具展示，不侵入核心聊天组件。
- **存储分层**：Settings / Sessions / ProviderKeys / CustomProviders 分 store 管理。
- **可替换网络层**：支持 CORS proxy 策略，适配浏览器环境。

这是“前端 Agent 平台化”的良好模板：组件化 + 插件化 + 状态持久化。

### 3.6 `packages/mom`（Slack 集成）

**定位**：把 coding-agent 能力接入 Slack 事件流。

关键设计点：

- **Channel 级运行状态**：每个频道独立 run state，支持 stop/working 指示。
- **上下文适配器模式**：把 Slack 事件映射到统一上下文接口（respond/replace/upload 等）。
- **异步串行更新**：通过更新队列避免并发消息覆盖。

适合借鉴到企业 IM（飞书/Teams/钉钉）Agent 网关设计。

### 3.7 `packages/pods`（推理部署与运维入口）

**定位**：GPU Pod 上 vLLM 等模型部署管理 CLI。

关键设计点：

- **模型配置与命令分离**：`models.json` 与命令逻辑分离，便于新增模型。
- **部署脚本化**：通过 scripts 固化环境初始化与模型启动步骤。
- **与 Agent 生态衔接**：为上层 Agent 提供可控后端推理基础设施。

如果你关注“自托管推理 + Agent 应用”，这层是成本与可控性的关键。

---

## 4. 关键跨包设计原则（最值得复用）

1. **统一消息协议优先于统一 SDK**
   - 先统一 `Message/ToolCall/ToolResult/Usage/StopReason`，再适配 provider SDK。
2. **状态机与展示层分离**
   - Agent loop 不依赖 TUI/Web，避免 UI 改动触发核心逻辑回归。
3. **工具能力 schema-first**
   - 工具定义、参数验证、执行结果结构化，提升稳定性与可测试性。
4. **扩展点前置设计**
   - provider 注册、tool renderer 注册、extension runner 都是插件点。
5. **长会话治理能力**
   - session 持久化、compaction、branching 是实战 Agent 的必要能力。

---

## 5. 推荐研究路径（两阶段）

### 阶段 A：建立“系统级心智模型”（1~2 天）

1. 通读仓库根 README，明确包边界与依赖方向。
2. 通读 `ai / agent / coding-agent / web-ui / tui` 各 README。
3. 画出你自己的“调用路径图”：
   - 用户输入 -> Agent loop -> LLM stream -> Tool -> ToolResult -> 回写 UI。

产出物建议：

- 一张架构图（Mermaid 或 draw.io）
- 一份术语表（Context、ToolCall、Compaction、Handoff 等）

### 阶段 B：做“可迁移的设计抽象”（3~5 天）

按下面顺序做源码深读：

1. `packages/ai/src/types.ts` + `stream.ts`（统一事件与 option 映射）
2. `packages/agent/src/agent.ts` + `agent-loop.ts`（核心状态推进）
3. `packages/coding-agent/src/core/*`（session、model、extensions、compaction）
4. `packages/web-ui/src/*`（AgentInterface、storage、tool renderers）
5. `packages/mom/src/main.ts`（外部事件系统适配）

产出物建议：

- “我自己的 Agent Runtime 抽象接口”草案
- “可替换模块清单”（模型层、工具层、存储层、UI 层）
- “技术债与风险清单”（例如 provider 差异、上下文膨胀、工具超时）

---

## 6. 面向你后续构建 Agent 应用的落地建议

### 6.1 最小可用技术栈组合

如果你要快速启动一个可用 Agent 应用：

- 模型层：复用 `pi-ai`
- 运行时：复用 `pi-agent-core`
- 交互层：
  - CLI 产品优先：参考 `pi-coding-agent`
  - Web 产品优先：参考 `pi-web-ui`

### 6.2 你的代码架构建议

建议建立以下目录边界（可类比本仓库）：

- `core/llm`：模型与 provider 适配
- `core/agent`：状态机、工具调度、会话治理
- `adapters/`：Slack/HTTP/WebSocket 等输入输出适配
- `ui/`：TUI 或 Web 组件
- `infra/`：部署、脚本、模型运维

### 6.3 先做稳定性，再做智能性

优先级建议：

1. 可恢复会话（持久化 + compaction）
2. 工具可靠执行（重试、超时、参数校验）
3. 可观测性（usage、cost、stop reason、trace）
4. 多模型策略（fallback、handoff）
5. 高级规划能力（plan mode、多代理协作）

---

## 7. 后续文档规划（建议你继续补齐）

建议以本文件为“总览”，继续拆出 4 份专题文档：

1. `docs/ai-layer-deep-dive.md`：重点写 provider 抽象和 stream 事件协议
2. `docs/agent-runtime-deep-dive.md`：重点写 agent loop 与工具执行事务
3. `docs/session-compaction-design.md`：重点写长会话治理策略
4. `docs/ui-integration-patterns.md`：重点写 TUI/Web/Slack 适配模式

这样能形成“总览 + 专题”的知识体系，后续你做新 Agent 产品时可以直接复用为架构蓝图。
