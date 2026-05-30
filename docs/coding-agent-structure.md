# coding-agent 目录结构详解

## 顶层文件

| 文件 | 作用 |
|------|------|
| `cli.ts` | 真正入口：设置 process title、配置 HTTP dispatcher、调用 `main()` |
| `main.ts` | CLI 参数解析总调度：读 stdin、选 session、解析 args，分发到 interactive/print/rpc 三种模式 |
| `config.ts` | 全局常量：`APP_NAME`、`VERSION`、目录路径（`~/.pi/`、`.pi/`）等 |
| `index.ts` | SDK 公开导出入口（供外部程序嵌入使用） |
| `migrations.ts` | 旧版配置格式迁移、废弃警告 |
| `package-manager-cli.ts` | `pi config` / `pi package` 子命令处理 |

---

## `src/cli/` — CLI 参数解析层

纯解析，不含业务逻辑。

| 文件 | 作用 |
|------|------|
| `args.ts` | CLI flag 定义与解析（`--model`、`--print`、`--rpc` 等），输出 `Args`/`Mode` 类型 |
| `file-processor.ts` | 处理 `--file` 参数，把文件内容转换为初始消息附件 |
| `initial-message.ts` | 组装首条用户消息（合并 prompt、管道 stdin、文件内容） |
| `list-models.ts` | `pi --list-models` 实现，格式化输出可用模型列表 |
| `session-picker.ts` | TUI 会话选择器（启动时选历史 session） |
| `config-selector.ts` | TUI 配置选择器 |

---

## `src/core/` — 核心业务逻辑

**中枢层，所有模式共享。**

### 会话管理

| 文件 | 作用 |
|------|------|
| `agent-session.ts` | **最核心文件**：`AgentSession` 类，持有单次对话完整状态——消息历史、工具列表、模型、compaction 状态、事件总线 |
| `agent-session-services.ts` | `createAgentSessionServices()`：组装 session 所需全部依赖（设置、扩展、工具、provider） |
| `agent-session-runtime.ts` | `beforeToolCall`/`afterToolCall` 运行时钩子：权限检查、工具调用拦截、重试逻辑 |
| `session-manager.ts` | 多 session 管理：fork、switch、clone、列出历史 session |
| `session-cwd.ts` | 工作目录校验：session 恢复时检测原目录是否还存在 |
| `event-bus.ts` | 内部事件总线，解耦 session 各子系统 |

### 设置与配置

| 文件 | 作用 |
|------|------|
| `settings-manager.ts` | 多层设置加载（global → project），支持热重载，管理 allowedTools/blockedTools 等权限 |
| `keybindings.ts` | `KeybindingsManager`：加载用户自定义快捷键覆盖 |
| `defaults.ts` | 默认配置值 |
| `resolve-config-value.ts` | 配置值解析（合并 global/project/runtime 三层优先级） |

### 模型

| 文件 | 作用 |
|------|------|
| `model-registry.ts` | 运行时模型注册表，聚合内置 + 扩展注册的模型 |
| `model-resolver.ts` | 按名称/别名解析具体 `Model` 对象，处理 scoped model（per-tool 模型指定） |
| `provider-display-names.ts` | Provider 名称的展示映射 |

### 认证

| 文件 | 作用 |
|------|------|
| `auth-storage.ts` | OAuth token 持久化存储（读写 `~/.pi/auth.json`） |
| `auth-guidance.ts` | 无可用模型时的认证引导提示文本 |

### `core/tools/` — Agent 可调用工具

每个文件对应一个工具。

| 文件 | 作用 |
|------|------|
| `bash.ts` | 在持久 shell 进程中执行命令，支持超时、输出截断、abort |
| `read.ts` | 读文件，支持行范围（`offset`/`limit`） |
| `write.ts` | 写文件（完整覆盖） |
| `edit.ts` | 精确字符串替换，保留上下文防止歧义 |
| `find.ts` | glob 文件查找，尊重 `.gitignore` |
| `grep.ts` | 正则内容搜索，返回匹配行 + 上下文 |
| `ls.ts` | 目录列表 |
| `file-mutation-queue.ts` | 文件写操作串行化队列，防止并发写冲突 |
| `edit-diff.ts` | Edit 操作的 diff 生成（用于 UI 展示） |
| `output-accumulator.ts` | 工具输出流式累积（bash 实时输出） |
| `truncate.ts` | 输出超长时截断策略 |
| `path-utils.ts` | 路径标准化工具 |
| `render-utils.ts` | 工具结果格式化渲染 |
| `tool-definition-wrapper.ts` | 工具定义包装器（添加元数据） |

### `core/extensions/` — 扩展系统

| 文件 | 作用 |
|------|------|
| `types.ts` | 扩展 API 类型：`ExtensionFactory`、`ExtensionContext`（`registerTool`/`registerEditor`/`on(event)`）、所有事件类型 |
| `loader.ts` | 扫描并加载 `.pi/extensions/*.ts`，动态 `import()` |
| `runner.ts` | 扩展生命周期：初始化、注入 context、触发事件钩子 |
| `wrapper.ts` | 扩展工具包装器，隔离扩展工具与内置工具 |

### `core/compaction/` — 上下文压缩

| 文件 | 作用 |
|------|------|
| `compaction.ts` | 超出上下文窗口时调 LLM 生成摘要替换旧消息 |
| `branch-summarization.ts` | fork/branch 场景下的分支摘要策略 |
| `utils.ts` | 压缩工具函数 |

### `core/export-html/` — HTML 导出

| 文件 | 作用 |
|------|------|
| `index.ts` | 导出入口，读 JSONL session 文件生成 HTML |
| `ansi-to-html.ts` | ANSI 转义序列 → HTML 颜色标签 |
| `tool-renderer.ts` | 各工具结果的 HTML 渲染 |

### 其他 core 文件

| 文件 | 作用 |
|------|------|
| `system-prompt.ts` | 动态组装系统提示词（CLAUDE.md + 资源 + 工具描述） |
| `skills.ts` | Skill 文件加载解析（markdown frontmatter + 内容） |
| `slash-commands.ts` | `/command` 注册与分发 |
| `prompt-templates.ts` | 系统提示词模板片段 |
| `resource-loader.ts` | MCP resource 加载（session 级资源注入） |
| `bash-executor.ts` | 底层 bash 执行器（独立于 bash 工具，供 RPC `bash` 命令用） |
| `http-dispatcher.ts` | 配置 undici 全局 HTTP dispatcher（代理、超时） |
| `output-guard.ts` | 拦截 `process.stdout`，防止工具输出污染 TUI 渲染 |
| `source-info.ts` | 代码位置信息（用于调试诊断） |
| `telemetry.ts` | 匿名使用统计 |
| `timings.ts` | 性能计时 |
| `diagnostics.ts` | 运行时诊断信息收集 |
| `footer-data-provider.ts` | 状态栏数据提供（token 数、耗时、模型名） |
| `sdk.ts` | SDK 公开接口（`createAgentSession` 等供外部嵌入） |
| `package-manager.ts` | 扩展包管理（安装/卸载 `.pi/extensions/` 中的包） |
| `exec.ts` | 子进程执行工具函数 |

---

## `src/modes/` — 运行模式

三种完全独立的 IO 模式，共享 `core/` 层。

### `interactive/` — TUI 交互模式

| 路径 | 作用 |
|------|------|
| `interactive-mode.ts` | 模式入口：初始化 TUI、挂载根组件、处理 resize |
| `components/assistant-message.ts` | 流式渲染 AI 回复（含 thinking、tool calls） |
| `components/user-message.ts` | 用户消息渲染 |
| `components/tool-execution.ts` | 工具执行状态渲染（running/done/error） |
| `components/diff.ts` | 文件编辑 diff 展示 |
| `components/footer.ts` | 状态栏（模型、token、耗时） |
| `components/model-selector.ts` | 运行时切换模型的 TUI 选择器 |
| `components/session-selector.ts` | 历史 session 浏览切换 |
| `components/tree-selector.ts` | 文件树选择器 |
| `components/login-dialog.ts` | OAuth 登录对话框 |
| `components/thinking-selector.ts` | thinking level 切换 |
| `components/extension-selector.ts` | 扩展选择/管理界面 |
| `components/settings-selector.ts` | 设置选择界面 |
| `components/skill-invocation-message.ts` | Skill 调用消息渲染 |
| `components/compaction-summary-message.ts` | 压缩摘要消息渲染 |
| `components/branch-summary-message.ts` | 分支摘要消息渲染 |
| `components/keybinding-hints.ts` | 底部快捷键提示条 |
| `theme/` | 主题系统：颜色方案定义、自动检测终端亮/暗色、文件监听热切换 |
| `assets/` | 内嵌静态资源 |

### `rpc/` — JSON-RPC 无头模式

| 文件 | 作用 |
|------|------|
| `rpc-types.ts` | 完整 RPC 协议类型：`RpcCommand`（stdin JSON lines）/ `RpcEvent`（stdout JSON lines）。涵盖 prompt、steer、follow_up、abort、模型控制、session 管理、compaction、bash 执行等全部命令 |
| `rpc-mode.ts` | RPC 模式入口：读 stdin → 解析命令 → 调 AgentSession → 输出事件流 |
| `rpc-client.ts` | RPC 客户端封装（供 SDK 消费者和测试用） |
| `jsonl.ts` | JSONL 行解析工具 |

### `print-mode.ts` — 非交互模式

单次运行，输出到 stdout，无 TUI。用于管道和脚本场景（`pi --print "..."`）。

---

## `src/utils/` — 通用工具

| 文件 | 作用 |
|------|------|
| `paths.ts` | 路径工具（tilde 展开、相对路径判断） |
| `syntax-highlight.ts` | highlight.js 代码高亮（终端内着色） |
| `git.ts` | git 状态查询（当前分支、diff 等） |
| `clipboard.ts` / `clipboard-native.ts` | 剪贴板读写（跨平台） |
| `image-convert.ts` / `image-resize.ts` | 图片转换与缩放（用于粘贴图片附件） |
| `fs-watch.ts` | 文件系统监听封装 |
| `shell.ts` | shell 命令执行工具函数 |
| `version-check.ts` | 检查 pi 是否有新版本 |
| `ansi.ts` | ANSI 序列处理工具 |
| `html.ts` | HTML 转义 |
| `frontmatter.ts` | YAML frontmatter 解析（skill 文件用） |
| `changelog.ts` | Changelog 解析 |
| `tools-manager.ts` | 工具允许/禁用列表管理 |
| `child-process.ts` | 子进程封装 |
| `mime.ts` | MIME 类型检测 |
| `exif-orientation.ts` | 图片 EXIF 方向修正 |
| `pi-user-agent.ts` | HTTP User-Agent 字符串生成 |
| `photon.ts` | 图片处理（photon-wasm 封装） |
| `sleep.ts` | Promise-based sleep |
| `windows-self-update.ts` | Windows 平台自更新逻辑 |

---

## `src/bun/` — Bun 运行时适配

| 文件 | 作用 |
|------|------|
| `cli.ts` | Bun 版入口（二进制打包用） |
| `register-bedrock.ts` | Bun 环境下注册 AWS Bedrock provider |
| `restore-sandbox-env.ts` | macOS 沙箱环境变量还原 |
