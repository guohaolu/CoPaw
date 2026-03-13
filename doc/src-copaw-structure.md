## src/copaw 目录结构总览

本笔记聚焦 `src/copaw` 这一层的项目结构，帮助从「整体 → 子模块 → 扩展点」系统理解 CoPaw 的架构设计，便于你在阅读源码或做二次开发时快速定位。

---

## 顶层模块一览

`src/copaw` 下的主要模块可以分成几大层：

- **启动与元信息层**
  - `__init__.py`：初始化日志和环境变量（从 `envs` 持久化存储中加载），是整个包的 bootstrap。
  - `__main__.py`：支持 `python -m copaw`，直接转到 CLI 入口。
  - `__version__.py`：版本号定义。
- **核心运行时 / Web 应用层**
  - `app/`：FastAPI 应用、AgentScope Runtime 集成、多通道消息接入、定时任务（cron）、MCP 管理等「运行时」能力。
  - `agents/`：围绕 AgentScope 的智能体封装（ReActAgent 扩展）、记忆管理、技能（skills）、工具（tools）、prompt 等。
- **配置与环境层**
  - `config/`：所有运行配置的 Pydantic 模型、加载与保存工具、配置文件路径管理、配置热更新 (ConfigWatcher)。
  - `constant.py`：工作目录、secret 目录、文件名常量、默认路径、环境变量 loader 等。
  - `envs/`：环境变量的持久化存储和加载逻辑（如从文件加载 env 到 `os.environ`）。
- **接入与交互层**
  - `cli/`：`copaw` 命令行的子命令实现（app 启动、channels 管理、cron、skills、models/providers、env、daemon 重启等）。
  - `app/channels/`：各类 IM / 协议通道（DingTalk、Feishu、QQ、Discord、iMessage、Telegram、MQTT、Console、Voice 等）的统一抽象与实现。
  - `website/console`（不在 `src/copaw` 内，但由 `app/_app.py` 挂载）：Web Console 静态资源。
- **模型与提供方层**
  - `providers/`：模型提供方管理（OpenAI、Anthropic、Ollama、ModelScope、DashScope 等），统一 ProviderManager。
  - `local_models/`：本地模型后端（llama.cpp、MLX 等）的抽象与工厂方法。
  - `tokenizer/`：分词器相关静态文件（vocab、merges 等）。
- **基础设施与工具层**
  - `utils/`：日志等通用工具。
  - `tunnel/`：Cloudflare 等隧道相关的辅助模块。

下面按目录逐层说明。

---

## app/：运行时与多通道接入

`app/` 是 CoPaw 的「应用层」，围绕 FastAPI + AgentScope Runtime 构建。

- **`app/_app.py`**
  - 创建 FastAPI 应用实例，并集成：
    - `AgentRunner`（对话执行、会话管理、自定义 QueryHandler）
    - `ChannelManager`（多通道消息接入）
    - `CronManager`（定时任务）
    - `MCPClientManager`（MCP 客户端管理）
    - `ConfigWatcher`、`MCPConfigWatcher`（配置/MCP 热更新）
  - 负责生命周期管理（`lifespan`）：启动阶段依次初始化 runner → MCP → channels → cron → chat manager → 各 watcher；关闭阶段按逆序释放资源。
  - 将 Console 静态资源 (`console/`) 作为前端挂载，同时暴露 `/api` 下的各类路由。

- **`app/runner/`：对话执行与会话管理**
  - `runner.py`：`AgentRunner` 继承 AgentScope Runtime 的 `Runner`，是「一个请求该怎么跑」的核心逻辑：
    - 判断是否命令（`/daemon ...`、`/compact` 等），命令路径下不创建 CoPawAgent，而是走命令 dispatcher。
    - 非命令路径下，实例化 `CoPawAgent`，加载会话状态（SafeJSONSession），在流式管线中执行推理并保存新状态。
  - `command_dispatch.py`：将字符串命令路由到：
    - Daemon 命令（`/daemon ...`）→ 重启、状态、logs 等。
    - 会话命令（`/compact`、`/new` 等）→ 使用轻量 `ReMeInMemoryMemory` + `CommandHandler` 操作记忆。
  - `daemon_commands.py`：Daemon 命令真正执行逻辑，并支持「进程内重启服务栈」。
  - `session.py`：`SafeJSONSession` 包装 AgentScope 的 JSONSession，在构造文件名时对 session_id/user_id 做跨平台安全清洗。
  - `query_error_dump.py`：当 query handler 抛异常时，落一份包含 traceback + request + agent_state 的 JSON 临时文件，便于排查。

- **`app/channels/`：多通道接入层**
  - `base.py`：定义 `BaseChannel` 抽象和 `ProcessHandler` 契约（`AgentRequest → Event` 流）。负责：
    - 原生 payload → AgentRequest 的转换钩子（由子类实现）。
    - 统一的内容渲染 (`MessageRenderer`) 和发送逻辑。
    - 「无文本」与时间防抖、会话级 debounce key 等通用机制。
  - `renderer.py`：`MessageRenderer` + `RenderStyle`，将 runtime Message 转为可发送的文本/媒体 parts，根据通道能力（是否 markdown/emoji）调整表现。
  - `manager.py`：`ChannelManager`：
    - 为每个通道维护 asyncio 队列和 worker，支持同一会话请求合并、不同会话并行处理。
    - 通过 `enqueue` 接收外部 payload，最终调用 `BaseChannel.consume_one`。
    - 支持动态替换单个 channel 实例（用于 config 热更新）。
  - `registry.py`：负责加载内建通道 & 工作目录下自定义通道（`custom_channels/`），并提供 `get_channel_registry()`。
  - `voice/`、`telegram/`、`dingtalk/`、`feishu/`、`qq/`、`discord_/`、`mqtt/`、`console/` 等：各通道的具体实现。

- **`app/crons/`：定时任务系统**
  - `manager.py`：`CronManager`，负责加载/保存 Job（基于 repo）、调度执行（结合 ChannelManager/Runner）。
  - `heartbeat.py`：心跳任务（如定时向指定通道发健康/状态信息）。
  - `repo/`：当前使用基于 JSON 文件的简单持久化实现。

- **`app/mcp/`：MCP 客户端与热更新**
  - `manager.py`：`MCPClientManager`，集中管理多个 MCP 客户端（HTTP/StdIO），支持 replace/remove/close_all。
  - `watcher.py`：`MCPConfigWatcher`，只关注 config 中的 MCP 片段变化，按 client 粒度热替换。

- **`app/routers/`：REST API 路由层**
  - `agent.py`：集中管理 agent 相关 HTTP API（如 memory 文件列表/读写等）。
  - `config.py`、`providers.py`、`skills.py`、`tools.py`、`workspace.py`、`envs.py`、`local_models.py`、`ollama_models.py`、`mcp.py`、`voice.py` 等：分别暴露管理配置、模型、技能、工作空间、环境变量、语音通道等的 REST 接口。

---

## agents/：AgentScope 集成与智能体封装

`agents/` 封装了 CoPaw 在 AgentScope 之上的「智能体层」能力。

- **`react_agent.py`**：`CoPawAgent`（继承 `ReActAgent`）
  - 初始化流程：
    - 构建内置工具 `Toolkit`（shell/file/browser 等），并根据 config 决定启用哪些工具。
    - 通过 `skills_manager` 动态加载工作目录下的 active skills（`active_skills/`），注册为 agent skill。
    - 构造系统提示词：从工作目录下的 `AGENTS.md` / `SOUL.md` / `PROFILE.md` 等文件构建，再叠加 env_context。
    - 创建模型与 formatter（通过 `model_factory` 封装 Provider）。
    - 设置记忆：默认 `InMemoryMemory`，如果有 `MemoryManager` 则用其 `get_in_memory_memory` 替换，并注册 memory_search tool。
    - 注册 hooks：`BootstrapHook`（首次交互引导）和 `MemoryCompactionHook`（上下文接近满时自动压缩记忆）。

- **`memory/`**
  - `memory_manager.py`：`MemoryManager` 继承 ReMeLight，负责：
    - 向量搜索 / 全文搜索的后端选择（local/chroma 等）。
    - 记忆压缩（compact_memory）与摘要（summary_memory），并支持工具辅助 summarize。
    - 提供 `get_in_memory_memory` 给 CoPawAgent 替换其 `memory`。
  - `agent_md_manager.py`：统一管理工作目录下的 `.md` 以及 `memory/` 目录的 `.md` 文件（列出、读写），方便通过 API/Console 操作长期记忆。

- **`hooks/`**
  - `bootstrap.py`：`BootstrapHook`，首条 user 消息前，根据 `BOOTSTRAP.md` 生成引导提示并插入用户消息后，将 `.bootstrap_completed` 标记文件写入，避免重复触发。
  - `memory_compaction.py`：`MemoryCompactionHook`，在每次推理前统计当前上下文 token 使用量，超过阈值时自动调用 `MemoryManager.compact_memory` 对旧消息压缩，并维护压缩标记。

- **`skills/` 与 `skills_manager.py`**
  - `skills/`：内置 skills 的目录结构，每个 skill 一个子目录，包含 `SKILL.md` + `scripts/`。
  - `skills_manager.py`：
    - 提供内置技能目录（代码内）、用户自定义技能目录（工作目录下 `customized_skills/`）、激活技能目录（`active_skills/`）的管理。
    - 支持将内置 + 自定义 skill 同步到 active 目录（可选择性/强制覆盖）。
    - 对单个 skill 的结构/内容读取（包括引用文件树与脚本文件树）。

- **`tools/`**
  - 将与文件、shell、浏览器、截图、时间、memory_search 等相关的操作封装成 AgentScope 的 Tool 函数，供 `CoPawAgent` 的 Toolkit 注册。

- **其它辅助模块**
  - `prompt/`：构建系统提示词（从工作目录 md 文件汇总）。
  - `model_factory.py`：基于 ProviderManager 创建统一的 ChatModel + Formatter。
  - `routing_chat_model.py`：根据配置在多个模型之间路由。
  - `command_handler.py`：处理系统命令 `/compact` `/new` `/clear` `/history` 等，对记忆与摘要进行操作。

---

## config/ 与 constant.py：配置与全局常量

- **`constant.py`**
  - 定义：
    - 工作目录 `WORKING_DIR`、密钥目录 `SECRET_DIR`、memory 目录、skills 目录、自定义通道目录等路径。
    - config 文件名、jobs 文件、chats 文件、heartbeat 文件等。
    - 加载这些配置的环境变量约定（如 `COPAW_WORKING_DIR` / `COPAW_CONFIG_FILE`）。

- **`config/config.py`**
  - 使用 Pydantic 定义整份 `config.json` 的数据结构：
    - `channels`：各通道的配置（token、开关、前缀等）。
    - `agents`：默认/运行时 agent 的配置（语言、max_iters、max_input_length、memory_compact_ratio 等）。
    - `mcp`：MCP 客户端配置。
    - `tools`：内置工具开关。
    - `last_api`、`last_dispatch` 等用于 Console 的辅助字段。

- **`config/utils.py`**
  - 获取 config 路径、加载/保存 config 的统一入口（`load_config` / `save_config`）。
  - 提供辅助函数获取 jobs/chats 路径等。

- **`config/watcher.py`**
  - `ConfigWatcher`：轮询 config 文件 mtime，仅在 channels 或 heartbeat 的 hash 变化时才执行：
    - 对通道：diff 前后配置，仅重建有变化的 channel（通过 `ChannelManager.replace_channel`）。
    - 对 heartbeat：若变更则调用 `CronManager.reschedule_heartbeat` 重新调度。

---

## cli/：命令行入口

- `main.py`：注册 `copaw` 根命令，以及 `app/channels/chats/cron/env/models/providers/skills/daemon/uninstall/desktop/clean` 等子命令。
- 各 `*_cmd.py`：对应一个子命令的实现，例如：
  - `app_cmd.py`：启动 Web app。
  - `init_cmd.py`：初始化工作目录与默认配置。
  - `channels_cmd.py`：列出/配置通道。
  - `skills_cmd.py`：交互式启用/禁用 skills。
  - `providers_cmd.py`：管理模型提供方。
  - `daemon_cmd.py`：通过 CLI 操作 daemon 状态/重启。

CLI 层主要是对 `app/` 与 `config/` 提供的一系列功能进行脚本化封装。

---

## providers/ 与 local_models/：模型提供方与本地模型

- **`providers/`**
  - `provider_manager.py`：`ProviderManager` 单例：
    - 管理内置 Provider（OpenAI、Anthropic、Ollama、ModelScope、DashScope、阿里云 CodingPlan 等）。
    - 支持从 `SECRET_DIR/providers/` 读取/迁移配置（旧版 `providers.json` → 多文件结构）。
    - 决定「某个逻辑模型槽」实际用哪个 provider/model。
  - `provider.py`、`openai_provider.py`、`anthropic_provider.py`、`ollama_provider.py` 等：具体 provider 的封装。

- **`local_models/`**
  - 定义本地模型 backend 抽象与实现（llama.cpp、MLX）；
  - 提供工厂函数用于从配置构建本地 ChatModel。

---

## envs/、utils/、tunnel/、tokenizer/：支撑模块

- **`envs/`**
  - 把用户在 Console/CLI 里配置的 env 持久化，并在 `copaw.__init__` 的早期加载到 `os.environ`。

- **`utils/logging.py`**
  - 统一日志配置（log level、file handler 等），供 CLI 与 app 复用。

- **`tunnel/`**
  - 目前主要与 Cloudflare 等隧道集成相关的辅助脚本，用于在某些部署场景下暴露本地服务。

- **`tokenizer/`**
  - 存放分词器文件（`vocab.json`、`merges.txt`、`tokenizer.json` 等），用于 token 计数或模型推理时的 tokenizer 支持。

---

## 学习建议：如何利用这套结构深入阅读

- **先理解「请求路径」**
  - IM → `app/channels/*` → `ChannelManager` → `AgentApp`/`AgentRunner.query_handler` → `CoPawAgent`/命令路径。
  - 配合 `app/routers/runner.py`、`app/runner/runner.py` 一起看，更容易形成完整的 call graph。

- **再看「Agent + 记忆 + 技能」**
  - `agents/react_agent.py`（CoPawAgent）如何组装 Toolkit/skills/memory/hook。
  - `agents/memory/` + `agents/hooks/` + `agents/skills_manager.py` 连起来看，理解记忆与技能如何「插入」到 AgentScope 流程里。

- **最后看「配置 + 热更新 + Daemon」**
  - `config/config.py`、`config/utils.py`、`config/watcher.py` 与 `app/_app.py` 中的 `ConfigWatcher`/`_do_restart_services`。
  - 加上 `app/runner/daemon_commands.py` 与 `/daemon restart` 的路径，可以看到这套结构如何支持「不停机调整配置」。

掌握了以上几个视角，再配合你自己的业务需求，就可以比较自信地在 CoPaw 之上做通道扩展、模型接入、Skills 扩展或 Agent 定制。 

