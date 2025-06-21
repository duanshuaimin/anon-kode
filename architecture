"anon-kode" 应用整体代码架构分析

"anon-kode" 是一个功能强大的命令行 AI 助手，使用 TypeScript 构建。它通过 Ink 库（React）渲染终端用户界面，并使用 Commander.js 解析命令行参数。其架构设计模块化，旨在高效管理人机交互、AI 模型通信、工具执行和配置。

1. 入口点与初始化 (src/entrypoints/cli.tsx)

应用从 cli.tsx 文件启动。
初始化 Sentry（错误跟踪）和 Statsig（功能开关）等服务。
参数解析: 使用 Commander.js 解析命令行参数（如 prompt, --cwd, --print）和子命令（如 config, mcp, log, resume）。
配置加载: 调用 src/utils/config.ts 中的 enableConfigs() 函数，以验证并允许读取全局和项目特定的配置。
设置 (setup() 函数): 处理当前工作目录设置、通过 src/context.ts 中的 getContext() 进行初始上下文预取、自动更新检查，并在需要时显示引导/信任对话框（通过 Onboarding, TrustDialog 组件）。
主执行流程:
如果使用 --print 参数并提供了提示，它会直接处理提示，将请求发送给 AI，打印响应后退出。
否则，它会渲染主 REPL 组件 (src/screens/REPL.tsx)，并传递可用的命令、工具和初始状态。
2. 核心交互 - REPL (src/screens/REPL.tsx)

REPL 组件是交互体验的核心。
状态管理: 管理关键状态，如对话 messages（消息列表）、isLoading（加载状态）、inputValue（输入框内容）、inputMode（输入模式：'prompt', 'bash', 'koding'）、abortController（用于取消请求）以及工具确认 (toolUseConfirm) 或自定义工具界面 (toolJSX) 的 UI 状态。
消息显示: 使用 Message 组件渲染对话历史。它会对消息进行规范化和记忆化处理以提高渲染效率。
用户输入 (PromptInput 组件):
捕获用户输入的文本。
通过 src/utils/messages.js 中的 processUserInput() 处理斜杠命令的检测和执行：
本地命令（如 /help, /config）会直接在客户端执行，可能会通过 setToolJSX 更新 UI 或修改消息。
基于提示的命令会为 AI 生成特定的提示。
对于直接发送给 AI 的提示或基于提示的命令，它会调用 onQuery 回调函数。
onQuery (在 REPL.tsx 中):
设置 isLoading 为 true，创建 AbortController。
将新的用户消息追加到 messages 状态中。
调用 src/query.ts 中的主 query() 函数与 AI 进行交互。
从 query() 流式接收响应，更新 messages 状态，从而触发 UI 重新渲染。
3. AI 交互 (src/query.ts, src/services/claude.ts, src/services/openai.ts)

query() (在 src/query.ts 中 - 异步生成器): 编排 AI 交互的生命周期。
提示组装: 接收当前 messages、基础 systemPrompt 数组和动态 context（来自 src/context.ts）。claude.ts 中的 formatSystemPromptWithContext() 会合并系统提示和动态上下文。
API 调用链: query.ts#query() -> claude.ts#queryOpenAI() (适配器) -> openai.ts#getCompletion()。
getCompletion() (在 openai.ts 中): 处理与配置的 OpenAI 兼容 API 端点的通信。管理 API 密钥（轮询、失败跟踪）、错误处理、重试逻辑和流式响应。
响应适配: claude.ts 中的 convertOpenAIResponseToAnthropic() 将 OpenAI API 的响应转换回内部使用的类 Anthropic 消息结构（例如 tool_use 块）。
工具处理:
如果 AI 响应包含 tool_use，query() 会调用 runToolUse()。
runToolUse() 验证输入，调用 REPL.tsx 中的 canUseTool()（如果需要，会触发 UI 权限请求），然后调用工具的 call() 方法。
工具结果被格式化后，query() 会递归调用自身并将这些结果传递回去。
4. 工具 (src/Tool.ts, src/tools/, src/tools.ts)

定义: 每个工具（如 BashTool）定义其 name（名称）、inputSchema（Zod 输入模式）、prompt()（给 AI 的指令）和 call()（执行逻辑）。还可以包含 UI 渲染方法。
发现 (src/tools.ts): getTools() 收集并筛选可用的工具。
执行: 由 query.ts 根据 AI 的 tool_use 请求触发。
5. 配置与上下文

应用配置 (src/utils/config.ts):
管理存储在中央 JSON 文件（例如 ~/.claude/config.json）中的全局和项目特定设置。
处理 API 密钥、模型偏好、UI 设置、部分工具权限以及项目特定的 AI 上下文片段。
支持 .mcprc 文件用于本地 MCP 服务器定义。
AI 上下文 (src/context.ts):
getContext() 为 AI 的系统提示动态组装上下文信息。
包括用户定义的上下文（来自配置）、推断的代码风格、目录结构、Git 状态、README.md 内容以及 KODING.md 文件列表。
6. 命令 (src/commands.ts, src/commands/)

斜杠命令（如 /help, /config）提供直接的用户控制功能。
类型：local（客户端代码执行）、local-jsx（客户端执行并渲染 UI）、prompt（生成 AI 提示）。
getCommands() 检索可用命令。processUserInput() 解析并分派这些命令。
典型的带工具使用的 AI 查询数据流:

用户在 REPL 中输入提示。
PromptInput -> processUserInput -> REPL.onQuery。
REPL.onQuery -> query.ts#query() (发送提示、历史记录、上下文)。
query.ts -> 适配器 (claude.ts, openai.ts) -> AI 模型 API。
AI 响应，其中包含 tool_use 请求。
响应被适配回 -> query.ts。
query.ts#runToolUse(): 识别工具，验证输入，通过 REPL 请求权限 (UI 对话框)。
用户授予权限。
query.ts -> 工具的 call() 方法执行。
工具 yield 结果 -> query.ts 将其格式化为 tool_result 消息。
query.ts 递归调用自身，并将 tool_result 消息加入历史。
AI 接收工具的输出。
AI 根据工具输出给出最终答案。
最终答案通过 query.ts 流式传输回 REPL 显示。
该架构使得 "anon-kode" 成为一个灵活且可扩展的 AI 助手，能够理解上下文、与各种 AI 模型交互，并利用工具在用户系统上执行操作。
