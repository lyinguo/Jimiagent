# MewCode

MewCode 是一个运行在终端里的 AI 编程助手。它可以读取和修改项目文件、执行命令、调用 MCP 工具、管理会话上下文，并通过权限系统控制写文件和命令执行等高风险操作。

这个项目的核心目标不是做一个简单的聊天壳子，而是把大模型、工具调用、上下文管理、权限控制和终端交互组织成一个可以在真实代码仓库中工作的 Agent。

## 主要能力

- 终端交互式编码助手，基于 Textual 构建 TUI 界面。
- 支持 Anthropic、OpenAI Responses API 和 OpenAI 兼容接口。
- 支持流式输出、工具调用、Thinking 输出和 token 用量统计。
- 内置文件读取、写入、编辑、搜索和 Bash 命令工具。
- 支持 MCP Server，把外部工具动态注册到 Agent。
- 支持权限模式，区分读取、写入和命令执行的审批策略。
- 支持上下文自动压缩，长对话接近 context window 时自动整理历史。
- 支持 Hook、Skill、子 Agent、Team、Worktree 等扩展能力。

## 快速开始

项目要求 Python 3.11 或更高版本。

```bash
pip install -e .
```

如果你使用 uv，也可以执行：

```bash
uv sync
```

启动交互式界面：

```bash
mewcode
```

执行一次性任务并把结果输出到终端：

```bash
mewcode -p "阅读这个项目，并总结它的架构"
```

也可以临时覆盖权限模式：

```bash
mewcode --mode plan
```

## 配置示例

MewCode 会按顺序读取并合并这些配置文件：

1. `~/.mewcode/config.yaml`
2. `./.mewcode/config.yaml`
3. `./.mewcode/config.local.yaml`

推荐把密钥放在环境变量中，项目配置里只写模型和接口信息：

```yaml
providers:
  - name: anthropic-official
    protocol: anthropic
    base_url: https://api.anthropic.com
    model: claude-sonnet-4-6
    thinking: true

mcp_servers:
  - name: context7
    command: npx
    args: ["-y", "@upstash/context7-mcp"]
```

OpenAI 兼容接口可以这样配置：

```yaml
providers:
  - name: openai-compatible
    protocol: openai-compat
    base_url: https://your-provider.example/v1
    model: your-model-name
    api_key: "${OPENAI_API_KEY}"
```

支持的 provider 协议：

- `anthropic`：Anthropic Messages API。
- `openai`：OpenAI Responses API。
- `openai-compat`：OpenAI 兼容的 Chat Completions API。

## 权限模式

`permission_mode` 用来控制 Agent 调用工具时是否需要确认：

| 模式 | 读取 | 写入 | 命令 |
| --- | --- | --- | --- |
| `default` | 允许 | 询问 | 询问 |
| `acceptEdits` | 允许 | 允许 | 询问 |
| `plan` | 允许 | 询问 | 询问 |
| `bypassPermissions` | 允许 | 允许 | 允许 |
| `custom` | 询问 | 询问 | 询问 |
| `dontAsk` | 允许 | 允许 | 允许 |

交互式使用时建议从 `default` 或 `plan` 开始；自动化脚本可以按需要使用 `dontAsk`。

## 常用命令

在 MewCode 的交互界面中，可以使用斜杠命令：

- `/help`：查看命令帮助。
- `/status`：查看当前状态。
- `/clear`：清空当前对话历史。
- `/compact`：手动压缩上下文。
- `/permission`：管理权限规则。
- `/memory`：管理记忆。
- `/mcp`：查看 MCP 服务器状态。
- `/skill`：管理 Skill 技能包。
- `/review`：审查代码变更。
- `/session`：管理会话。
- `/worktree`：管理 Git worktree。
- `/tasks`：管理后台任务。
- `/trace`：查看 Agent 追踪树。

## 这段代码做了什么

从入口看，`mewcode.__main__:main` 是整个程序的启动点：

1. 创建 `.mewcode/` 目录并把运行日志写入 `.mewcode/debug.log`。
2. 解析命令行参数，例如 `--mode` 和 `-p`。
3. 加载配置文件，得到 provider、MCP、Hook、权限和 worktree 设置。
4. 根据权限模式创建权限检查器。
5. 如果传入 `-p`，进入非交互模式，创建 Agent 并运行一次任务。
6. 如果没有传入 `-p`，启动 Textual TUI 应用，让用户在终端中持续对话。

核心执行链路大致如下：

```text
用户输入
  -> ConversationManager 记录对话
  -> Agent 构造 system prompt 和工具 schema
  -> LLMClient 调用模型并接收流式响应
  -> 模型请求工具调用
  -> PermissionChecker 判断是否允许执行
  -> ToolRegistry 找到并执行对应工具
  -> 工具结果写回对话
  -> Agent 继续下一轮，直到模型给出最终答案
```

其中 `mewcode/client.py` 负责屏蔽不同模型接口的差异。无论底层是 Anthropic、OpenAI Responses API，还是 OpenAI 兼容接口，它都会把模型输出统一转换成这些事件：

- 文本增量输出。
- Thinking 增量输出。
- 工具调用开始、参数增量和调用完成。
- 一轮响应结束和 token 用量统计。

`mewcode/agent.py` 是主循环。它会处理系统提示、长期记忆、上下文压缩、Hook、权限审批、工具执行、错误重试和最终输出。可以把它理解成 MewCode 的调度中心。

## 项目结构

下面按目录介绍主要文件。`__pycache__`、运行时日志和本地配置等生成文件不列入说明。

### 根目录

```text
README.md              项目说明文档，介绍安装、配置、能力和代码结构
MEWCODE.md             项目级给 Agent 读取的长期指令/工作约定
pyproject.toml         Python 包元数据、依赖、命令入口和测试配置
uv.lock                uv 依赖锁文件，用于复现依赖版本
.gitignore             Git 忽略规则，排除虚拟环境、运行缓存、本地配置等文件
mewcode-python.zip     项目源码压缩包或发布辅助包
.mewcode/              本地运行目录，保存 config.yaml、debug.log、会话和文件历史；通常不提交
.venv/                 本地 Python 虚拟环境；由依赖管理工具创建，通常不提交
```

### `mewcode/` 核心包

```text
mewcode/
  __init__.py          包初始化文件，暴露版本或包级入口
  __main__.py          CLI 入口；解析参数、加载配置、注册工具并启动交互/非交互模式
  agent.py             Agent 主循环；处理模型流、工具调用、权限、Hook、压缩、记忆和通知
  app.py               Textual TUI 应用；负责终端界面、输入输出、权限弹窗、任务通知和命令注册
  askuser_dialog.py    用户提问弹窗，用于工具需要向用户确认或收集信息时展示 UI
  cache.py             通用缓存辅助逻辑，供上下文或工具结果复用
  client.py            LLM 客户端适配层；统一 Anthropic、OpenAI Responses 和 OpenAI 兼容接口事件
  config.py            配置模型和加载逻辑；合并全局/项目/本地配置并解析 provider、worktree 等选项
  conversation.py      对话历史、消息、tool use/tool result/thinking 等结构定义
  driver.py            程序驱动辅助逻辑，连接 Agent 执行和外层运行方式
  permission_dialog.py 权限审批弹窗；展示工具风险并收集允许/拒绝/总是允许
  plan_dialog.py       Plan 模式审批 UI；用于展示计划并让用户确认是否继续执行
  prompts.py           系统提示词、环境上下文、计划模式提醒和 coordinator prompt 的构建逻辑
  serialization.py     把内部对话结构转换成不同模型协议需要的消息格式
  session_dialog.py    会话管理弹窗；展示、恢复或切换历史会话
  styles.tcss          Textual 样式表，定义终端 UI 的布局、颜色和组件样式
  teammate_tree.py     团队/队友树形展示组件，辅助显示多 Agent 协作状态
  validator.py         配置结构校验，保证 config.yaml 的字段和类型符合预期
```

### `mewcode/agents/` 子 Agent 和后台任务

```text
mewcode/agents/
  __init__.py          子 Agent 包导出入口
  fork.py              Fork 模式；把父对话复制成子 Agent 上下文并追加 fork 任务说明
  loader.py            Agent 定义加载器；按项目级、用户级、内置顺序加载 `.md` 子 Agent 定义
  notification.py      后台任务完成通知的格式化和注入逻辑
  parser.py            解析子 Agent `.md` frontmatter，生成 AgentDef 配置对象
  task_manager.py      后台任务管理器；启动、取消、轮询子 Agent 和 team member 的异步任务
  tool_filter.py       根据 AgentDef 过滤工具；控制子 Agent、后台任务、队友和协调者可用工具
  trace.py             Agent 追踪树；记录父子关系、token 用量、状态和耗时
```

#### `mewcode/agents/builtins/` 内置子 Agent 定义

```text
mewcode/agents/builtins/
  __init__.py          内置定义包标记文件
  explore.md           只读探索型子 Agent，适合搜索代码、理解结构和调用链
  general-purpose.md   通用型子 Agent，适合独立完成研究、实现或综合任务
  plan.md              规划型子 Agent，只读分析需求并输出实现计划
  verification.md      验证型子 Agent，只读运行测试/构建/检查并给出 VERDICT
```

### `mewcode/commands/` 斜杠命令系统

```text
mewcode/commands/
  __init__.py          命令包导出入口
  completion.py        命令补全逻辑，支持 TUI 输入时补全斜杠命令或参数
  parser.py            解析用户输入中的斜杠命令和参数
  registry.py          命令注册表和 Command/CommandContext 定义
```

#### `mewcode/commands/handlers/` 命令处理器

```text
mewcode/commands/handlers/
  __init__.py          命令处理器包导出入口
  clear.py             `/clear`，清空当前对话历史
  compact.py           `/compact`，手动触发上下文压缩
  help.py              `/help`，展示可用命令和使用方式
  mcp.py               `/mcp`，查看 MCP server 状态和已注册工具
  memory.py            `/memory`，查看、写入或管理长期记忆
  permission.py        `/permission`，查看或调整权限规则
  plan.py              `/plan`，进入或退出计划模式
  review.py            `/review`，触发代码审查工作流
  rewind.py            `/rewind`，回退会话历史到较早位置
  session.py           `/session`，管理和恢复历史会话
  skill.py             `/skill`，列出、加载或管理 Skill
  skill_register.py    注册 Skill 相关命令或辅助入口
  status.py            `/status`，展示 provider、权限、token、工作目录等运行状态
  tasks.py             `/tasks`，查看和管理后台 Agent 任务
  trace.py             `/trace`，展示 Agent 追踪树和 token 汇总
  worktree.py          `/worktree`，创建、进入、退出和查看 Git worktree
```

### `mewcode/context/` 上下文管理

```text
mewcode/context/
  __init__.py          上下文模块导出入口
  manager.py           自动压缩、压缩边界、工具结果预算和内容替换状态管理
```

### `mewcode/filehistory/` 文件历史

```text
mewcode/filehistory/
  __init__.py          文件历史模块导出入口
  history.py           记录文件读取/编辑历史，辅助检测文件状态变化和回溯
```

### `mewcode/hooks/` Hook 系统

```text
mewcode/hooks/
  __init__.py          Hook 包导出入口
  conditions.py        Hook 条件匹配逻辑，例如按工具名、事件类型或路径过滤
  engine.py            Hook 执行引擎；在 turn/tool 等事件点运行配置好的 Hook
  events.py            Hook 事件名称和事件数据定义
  executors.py         Hook 执行器；支持命令、脚本或其他动作的实际执行
  loader.py            从配置加载 Hook 定义并转换为内部模型
  models.py            Hook 配置、条件、动作等数据模型
```

### `mewcode/mcp/` MCP 集成

```text
mewcode/mcp/
  __init__.py          MCP 模块导出入口
  client.py            MCP 客户端；启动外部 MCP server 并进行协议通信
  manager.py           MCP server 生命周期管理和工具注册协调
  tool_wrapper.py      把 MCP 工具包装成 MewCode 内部 Tool，统一暴露给 Agent 调用
```

### `mewcode/memory/` 记忆和项目指令

```text
mewcode/memory/
  __init__.py          记忆模块导出入口
  auto_memory.py       自动记忆抽取和 MemoryManager，定期从对话中提炼长期信息
  instructions.py      加载项目/用户指令文件，例如 MEWCODE.md 或类似约定
  recall.py            记忆检索逻辑，按当前任务召回相关长期记忆
  session.py           会话级记忆或状态持久化辅助逻辑
```

### `mewcode/permissions/` 权限和沙箱

```text
mewcode/permissions/
  __init__.py          权限模块导出入口
  checker.py           PermissionChecker；统一判断工具调用是否允许、询问或拒绝
  dangerous.py         危险命令检测器；识别删除、重置、破坏性 git/系统命令等风险
  modes.py             权限模式枚举和默认策略，例如 default、acceptEdits、dontAsk
  rules.py             权限规则引擎；管理用户允许/拒绝/总是允许的规则
  sandbox.py           路径沙箱；限制读写范围，防止越权访问工作区外路径
```

### `mewcode/skills/` Skill 技能系统

```text
mewcode/skills/
  __init__.py          Skill 包导出入口
  directory.py         Skill 目录发现和路径解析，支持内置、项目级和用户级技能
  executor.py          Skill 执行器；把已加载 Skill 的说明注入 Agent 或执行相关动作
  loader.py            加载 Skill 定义、缓存目录和可用技能列表
  parser.py            解析 SKILL.md 和可选元数据文件
```

#### `mewcode/skills/builtins/` 内置 Skill

```text
mewcode/skills/builtins/
  __init__.py                          内置 Skill 包标记文件
  backend-interview/
    __init__.py                        后端面试 Skill 包标记文件
    SKILL.md                           后端面试辅助 Skill 的说明、流程和提示词
    tool.json                          该 Skill 关联工具或元数据配置
    references/parse_resume.py         简历解析参考脚本或示例工具
  commit/
    __init__.py                        commit Skill 包标记文件
    SKILL.md                           生成提交信息、整理变更摘要的 Skill 说明
  review/
    __init__.py                        review Skill 包标记文件
    SKILL.md                           代码审查流程、关注点和输出格式说明
  test/
    __init__.py                        test Skill 包标记文件
    SKILL.md                           运行测试、分析失败和汇报结果的 Skill 说明
```

### `mewcode/teams/` 多 Agent 团队协作

```text
mewcode/teams/
  __init__.py          Team 包导出入口
  backend_detect.py    队友运行后端检测；当前默认使用 in-process，也保留 tmux/iTerm2 检测逻辑
  coordinator.py       协调者模式 prompt；指导主 Agent 拆分任务、调度 worker 和汇总结果
  mailbox.py           文件邮箱；用 JSON 文件在 lead 和队友之间传递消息
  manager.py           TeamManager；创建/删除团队、注册成员、管理 mailbox、任务板和 worktree
  models.py            AgentTeam、TeammateInfo、BackendType 等团队数据模型
  progress.py          队友进度跟踪；记录工具调用、token 和最近输出，用于 UI 展示
  registry.py          Agent 名称注册表；把队友名字解析到 agent_id
  shared_task.py       团队共享任务板；管理任务创建、查询、状态和依赖关系
  spawn_inprocess.py   进程内队友启动逻辑；用 asyncio task 运行队友 Agent
  spawn_iterm2.py      iTerm2 pane 队友启动逻辑
  spawn_tmux.py        tmux pane 队友启动、唤醒和终止逻辑
  transcript.py        团队成员对话记录保存和恢复
```

### `mewcode/tools/` 内置工具

```text
mewcode/tools/
  __init__.py          ToolRegistry 和内置工具导出入口
  agent_tool.py        `Agent` 工具；启动普通子 Agent、fork Agent、worktree Agent 或 team member
  ask_user.py          `AskUserQuestion` 工具；让模型向用户提出结构化问题
  base.py              Tool 基类、ToolResult、流式事件和工具输出限制等基础定义
  bash.py              `Bash` 工具；执行 shell 命令并接入权限和危险命令检测
  edit_file.py         `EditFile` 工具；对现有文件做精确编辑
  enter_worktree.py    `EnterWorktree` 工具；创建隔离 Git worktree 并切换会话工作目录
  exit_plan_mode.py    `ExitPlanMode` 工具；提交计划并请求用户批准执行
  exit_worktree.py     `ExitWorktree` 工具；退出 worktree 会话并选择保留或删除
  file_state_cache.py  文件状态缓存；辅助编辑前后检查文件是否变化
  glob.py              `Glob` 工具；按模式查找文件路径
  grep.py              `Grep` 工具；搜索文件内容
  load_skill.py        `LoadSkill` 工具；加载指定 Skill 并加入当前上下文
  read_file.py         `ReadFile` 工具；读取文件内容并处理范围、大小和编码
  send_message.py      `SendMessage` 工具；team member 之间或队友与 lead 之间发消息
  synthetic_output.py  `SyntheticOutput` 工具；让 coordinator 输出结构化结果或状态
  task_create.py       `TaskCreate` 工具；在团队共享任务板创建任务
  task_get.py          `TaskGet` 工具；读取单个团队任务详情
  task_list.py         `TaskList` 工具；列出或筛选团队任务
  task_update.py       `TaskUpdate` 工具；更新任务状态、负责人和依赖
  team_create.py       `TeamCreate` 工具；创建多 Agent 团队并可激活协调者模式
  team_delete.py       `TeamDelete` 工具；销毁 idle 团队、清理邮箱、worktree 和 pane
  write_file.py        `WriteFile` 工具；写入或创建文件
```

#### `mewcode/tools/impl/` 工具内部实现

```text
mewcode/tools/impl/
  __init__.py          工具实现子包标记文件
  tool_search.py       工具搜索实现；用于在大量可用工具中查找匹配工具
```

### `mewcode/worktree/` Git worktree 隔离

```text
mewcode/worktree/
  __init__.py          Worktree 包导出入口
  changes.py           检测 worktree 未提交变更、新提交和未推送提交
  cleanup.py           清理过期临时 worktree，避免长期积累无用目录和分支
  integration.py       给 worktree 子 Agent 注入路径隔离说明
  manager.py           WorktreeManager；创建、进入、退出、恢复和删除 Git worktree
  models.py            Worktree 和 WorktreeSession 数据模型
  session.py           worktree 会话持久化，支持程序重启后恢复当前工作区
  setup.py             worktree 创建后的环境准备；复制本地配置、设置 hooks、软链依赖目录
  slug.py              worktree 名称校验和路径安全转换
```

### `tests/` 测试

```text
tests/
  test_agent.py             Agent 主循环、工具调用和事件处理测试
  test_commands.py          斜杠命令注册、解析和处理器测试
  test_context.py           上下文压缩和工具结果预算测试
  test_context_window.py    context window 计算和阈值相关测试
  test_hooks.py             Hook 加载、条件匹配和执行测试
  test_mcp.py               MCP 客户端、管理器和工具包装测试
  test_memory.py            记忆加载、合并、召回和自动提取测试
  test_permissions.py       权限模式、规则、沙箱和危险命令检测测试
  test_recovery.py          错误恢复、压缩恢复或会话恢复相关测试
  test_replacement_state.py 内容替换状态和压缩边界重建测试
  test_serialization.py     Anthropic/OpenAI 等协议消息序列化测试
  test_skills.py            Skill 发现、解析、加载和执行测试
  test_subagent.py          子 Agent 定义、工具过滤、fork、后台任务和通知测试
  test_teams.py             Team、mailbox、共享任务、队友和协调者模式测试
  test_tool_search.py       工具搜索功能测试
  test_worktree.py          Git worktree 创建、进入、退出、清理和变更检测测试
  verify_subagent.py        子 Agent 系统端到端验证脚本，偏集成验证用途
```

## 开发与测试

运行测试：

```bash
pytest
```

只检查 Python 文件是否能正常编译：

```bash
python -m py_compile mewcode/*.py
```

## 注意事项

- 不建议把真实 API Key 直接提交到仓库，优先使用环境变量或本地私有配置文件。
- `bypassPermissions` 和 `dontAsk` 会自动允许写入和命令执行，适合受控环境，不适合不可信仓库。
- MCP Server 可能需要网络或本机运行时依赖，例如 `npx`。
