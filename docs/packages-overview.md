# packages 各包功能概览

## 依赖链

```
tui (终端渲染)
  ↓
ai (LLM 抽象层)
  ↓
agent (循环 + 工具执行)
  ↓
coding-agent (pi CLI = 工具 + 会话 + UI + 扩展)
```

---

## packages/ai — LLM 提供商抽象层

**核心职责：** 统一封装多个 LLM 提供商，对外暴露一致的流式 API。


| 文件                                             | 功能                                                                                                                                                    |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/types.ts`                                 | 全局核心类型：`Api`/`Provider`/`Model`、`StreamOptions`、`Message`（User/Assistant/ToolResult）、`AssistantMessageEvent` 流事件协议、兼容层配置（`OpenAICompletionsCompat` 等） |
| `src/providers/anthropic.ts`                   | Anthropic Messages API 实现，支持 eager tool streaming、cache control                                                                                       |
| `src/providers/openai-completions.ts`          | OpenAI Completions 兼容实现，含大量 compat 标志兼容不同厂商                                                                                                           |
| `src/providers/openai-responses.ts`            | OpenAI Responses API 实现                                                                                                                               |
| `src/providers/google.ts` / `google-vertex.ts` | Google Generative AI / Vertex AI 实现                                                                                                                   |
| `src/providers/amazon-bedrock.ts`              | AWS Bedrock Converse Stream 实现                                                                                                                        |
| `src/providers/mistral.ts`                     | Mistral Conversations API 实现                                                                                                                          |
| `src/providers/faux.ts`                        | 测试用假 provider，不发真实请求                                                                                                                                  |
| `src/providers/register-builtins.ts`           | 懒加载所有内置 provider，不能静态 import 以避免打包体积问题                                                                                                                |
| `src/models.generated.ts`                      | 自动生成的模型列表（严禁手动编辑）                                                                                                                                     |
| `src/api-registry.ts`                          | 全局 API 注册表，provider 按需注册                                                                                                                              |
| `src/stream.ts`                                | `streamSimple()` 统一入口，路由到对应 provider                                                                                                                  |
| `src/env-api-keys.ts`                          | 环境变量 → 凭证检测映射                                                                                                                                         |
| `src/oauth.ts`                                 | OAuth 登录流程（Anthropic、GitHub Copilot、OpenAI Codex）                                                                                                     |
| `src/images.ts`                                | 图像生成统一 API                                                                                                                                            |
| `src/utils/event-stream.ts`                    | `AssistantMessageEventStream` 异步可迭代流实现                                                                                                                |


---

## packages/agent — 提供商无关的 Agent 循环

**核心职责：** 在 `pi-ai` 之上实现工具调用循环、消息历史管理、事件驱动的 agent 状态机。


| 文件                             | 功能                                                                                                                                             |
| ------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------- |
| `src/agent-loop.ts`            | 核心循环：`agentLoop()`（新 prompt）/ `agentLoopContinue()`（续跑）。内部双层 while 循环：外层处理 follow-up，内层处理 tool 调用和 steering 消息。工具执行支持 sequential/parallel 两种模式 |
| `src/agent.ts`                 | 高层 `Agent` 类，封装状态（`isStreaming`、`pendingToolCalls`、`messages`）和 `subscribe()` 事件订阅                                                             |
| `src/types.ts`                 | `AgentLoopConfig`（含 `beforeToolCall`/`afterToolCall`/`shouldStopAfterTurn` 等钩子）、`AgentTool`、`AgentEvent` 联合类型、`AgentMessage` 可扩展消息类型           |
| `src/harness/agent-harness.ts` | 测试 harness，用 faux provider 驱动 agent，不消耗真实 API token                                                                                            |
| `src/harness/session/`         | Session 的 JSONL 持久化（`jsonl-repo.ts`）和内存存储（`memory-repo.ts`）实现                                                                                  |
| `src/harness/compaction/`      | 消息压缩/分支摘要，用于超出上下文窗口时的历史截断                                                                                                                      |
| `src/harness/skills.ts`        | Skill 加载与解析                                                                                                                                    |
| `src/harness/system-prompt.ts` | System prompt 模板组装                                                                                                                             |
| `src/proxy.ts`                 | HTTP 代理支持                                                                                                                                      |


---

## packages/tui — 终端 UI 原语库

**核心职责：** 提供差量渲染 TUI 引擎和可组合 UI 组件，供 `coding-agent` 的交互模式使用。


| 文件                                                   | 功能                                                                                   |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `src/tui.ts`                                         | TUI 引擎核心：`Component` 接口定义（`render(width)`/`handleInput()`）、差量渲染（只重绘变更行）、Kitty 图像协议支持 |
| `src/terminal.ts`                                    | 底层终端控制：光标移动、清屏、ANSI 序列                                                               |
| `src/stdin-buffer.ts`                                | stdin 字节缓冲与事件分发                                                                      |
| `src/keys.ts`                                        | 键码解析，支持 Kitty 增强键盘协议                                                                 |
| `src/keybindings.ts`                                 | `DEFAULT_EDITOR_KEYBINDINGS`/`DEFAULT_APP_KEYBINDINGS`，所有按键必须经此配置，禁止硬编码键字符串          |
| `src/kill-ring.ts`                                   | Emacs 风格 kill-ring，支持多次剪切拼接粘贴                                                        |
| `src/undo-stack.ts`                                  | 编辑撤销栈                                                                                |
| `src/fuzzy.ts`                                       | 模糊搜索（用于文件/命令选择）                                                                      |
| `src/autocomplete.ts`                                | 自动补全逻辑                                                                               |
| `src/terminal-image.ts`                              | Kitty graphics protocol 图像渲染                                                         |
| `src/components/editor.ts`                           | 多行文本编辑器组件                                                                            |
| `src/components/input.ts`                            | 单行输入框                                                                                |
| `src/components/markdown.ts`                         | Markdown 渲染（终端内）                                                                     |
| `src/components/select-list.ts`                      | 可导航列表选择                                                                              |
| `src/components/loader.ts` / `cancellable-loader.ts` | 加载动画                                                                                 |
| `src/components/box.ts`                              | 带边框容器                                                                                |


---

## packages/coding-agent — `pi` CLI 二进制

**核心职责：** 最高层包，将上面三个包组合为用户可用的 AI 编码助手 CLI。


| 路径                                                 | 功能                                                  |
| -------------------------------------------------- | --------------------------------------------------- |
| `src/cli.ts`                                       | 入口点：设置 process title、配置 HTTP dispatcher、调用 `main()` |
| `src/main.ts`                                      | CLI 参数解析，分发到各运行模式                                   |
| `src/core/agent-session.ts`                        | **中枢**：持有单个对话会话状态，集成 agent loop、工具、扩展、compaction、设置 |
| `src/core/agent-session-runtime.ts`                | 运行时钩子：beforeToolCall/afterToolCall 权限检查、工具执行拦截      |
| `src/core/settings-manager.ts`                     | 多层设置加载（global `~/.pi/` → project `.pi/`），支持热重载      |
| `src/core/system-prompt.ts`                        | 动态组装系统提示词（含 CLAUDE.md、资源、工具描述）                      |
| `src/core/tools/bash.ts`                           | Bash 工具：在持久 shell 中执行命令，支持超时和输出截断                   |
| `src/core/tools/read.ts`                           | 文件读取工具，支持行范围                                        |
| `src/core/tools/write.ts`                          | 文件写入工具                                              |
| `src/core/tools/edit.ts`                           | 精确字符串替换编辑工具                                         |
| `src/core/tools/find.ts`                           | glob 文件查找                                           |
| `src/core/tools/grep.ts`                           | 正则内容搜索                                              |
| `src/core/tools/ls.ts`                             | 目录列表                                                |
| `src/core/compaction/compaction.ts`                | 上下文压缩：当消息超出窗口时调用 LLM 生成摘要                           |
| `src/core/extensions/loader.ts`                    | 扩展文件加载（`.pi/extensions/*.ts`）                       |
| `src/core/extensions/runner.ts`                    | 扩展生命周期管理，注入 `ExtensionContext`                      |
| `src/core/extensions/types.ts`                     | 扩展 API 类型：`registerTool`/`registerEditor`/事件钩子      |
| `src/core/skills.ts`                               | Skill 文件加载与解析（`~/.pi/skills/` 或 `.pi/skills/`）      |
| `src/core/slash-commands.ts`                       | `/command` 解析与分发                                    |
| `src/core/session-manager.ts`                      | 多会话管理（fork、switch、列表）                               |
| `src/core/model-registry.ts` / `model-resolver.ts` | 模型注册与解析（按名称/别名查找 Model 对象）                          |
| `src/modes/interactive/`                           | TUI 交互模式：渲染聊天界面，处理用户输入                              |
| `src/modes/print-mode.ts`                          | 非交互 print 模式（管道/脚本用）                                |
| `src/modes/rpc/`                                   | JSON-RPC 模式，programmatic 控制（完整协议见 `rpc-types.ts`）   |
| `src/core/export-html/`                            | 对话导出为 HTML                                          |


