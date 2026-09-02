# MewCode

> 一款面向本地代码仓库的终端 AI 编程助手。它将模型的推理与受控的文件、搜索、Shell、MCP 和多 Agent 能力结合起来，帮助你完成编码、排错、重构与代码探索。

MewCode 不只是把聊天窗口搬进终端。它以当前项目为工作区，让 Agent 能够理解仓库上下文、调用本地工具，并在执行写入或命令前遵循可配置的权限策略。

- **终端交互界面**：基于 Textual 的对话式 TUI，支持流式输出、工具调用展示、命令补全、输入历史与 `@文件` 引用。
- **可编程的代码操作**：内置读取、写入、精确编辑、Glob、Grep 和 Shell 工具；读取同一文件时会复用缓存，降低重复开销。
- **多模型接入**：支持 Anthropic Messages API、OpenAI Responses API，以及兼容 OpenAI Chat Completions 的服务。
- **权限与安全边界**：为读、写、命令分别提供权限模式、危险命令检测、工作目录沙箱和可持久化规则。
- **长任务可持续推进**：自动压缩上下文；支持会话恢复、文件检查点回退和长期记忆。
- **可扩展的 Agent 工作流**：支持 Skills、Hooks、MCP 工具、后台子 Agent、可通信的 Agent 团队和独立 Git worktree。

<!-- TODO: Add an architecture or feature-overview image here. Example: ![MewCode architecture](docs/images/architecture.png) -->

## 功能一览

| 能力 | 说明 |
| --- | --- |
| 交互与自动化 | 默认启动为终端交互界面；`-p` 可将提示词作为一次性任务执行，便于脚本或 CI 调用。 |
| 仓库理解 | 可搜索、读取与引用项目文件；项目说明文件会被加载为 Agent 指令。 |
| 编辑与回退 | Agent 可写入或精确编辑文件；每轮产生文件检查点，可选择恢复代码、对话，或两者同时恢复。 |
| 上下文管理 | 接近模型上下文上限时自动压缩；也可通过 `/compact` 手动压缩，并保留必要的环境、记忆和关键文件信息。 |
| 会话与记忆 | 会话可创建、列出、恢复、删除；退出时可从对话中提取用户偏好、纠正反馈、项目知识与参考资料。 |
| 权限控制 | 提供默认、自动接受编辑、计划、完全放行、自定义等模式；规则可分用户级、项目级和本地级管理。 |
| Skills | 从内置、用户级或项目级 `SKILL.md` 发现技能，并支持按需加载、热重载和受限工具集。 |
| Hooks | 在启动、会话、轮次、工具调用、压缩等生命周期事件上执行命令、提示词、HTTP 或 Agent 动作；前置 Hook 可阻止工具调用。 |
| MCP | 连接 stdio 或 HTTP MCP Server，并将远程能力注册为工具。 |
| 子 Agent 与团队 | 提供探索、规划、通用执行、验证等内置角色；可追踪子 Agent，并让团队成员通过邮箱式消息协作。 |
| Git Worktree | 为隔离任务创建和进入 Git worktree；退出时会检查未提交改动或新增提交，避免误删工作成果。 |

## 快速开始

### 1. 前置条件

- Python **3.11+**
- 一个可用的 Anthropic、OpenAI 或 OpenAI 兼容服务的 API Key
- 可选：Git（使用 Worktree 隔离任务时需要）
- 推荐安装 [uv](https://docs.astral.sh/uv/) 管理项目环境

### 2. 安装依赖

```bash
git clone <your-repository-url>
cd mewcode-python
uv sync
```

也可以使用 pip：

```bash
python -m venv .venv
.venv/bin/pip install -e .
```

Windows PowerShell 请将最后一行替换为：

```powershell
.\.venv\Scripts\pip install -e .
```

### 3. 配置模型提供商

在项目根目录创建 `.mewcode/config.yaml`。密钥既可以写在 `api_key` 中，也可以通过环境变量提供；以下示例使用 `OPENAI_API_KEY`。

```yaml
providers:
  - name: OpenAI
    protocol: openai
    base_url: https://api.openai.com/v1
    model: gpt-4.1

# 可选：default、acceptEdits、plan、bypassPermissions、custom、dontAsk
permission_mode: default

# 可选：开启对话上下文继承式的子 Agent
enable_fork: true

# 可选：为子 Agent 启用只读验证角色
enable_verification_agent: true
```

设置环境变量后启动：

```bash
# macOS / Linux
export OPENAI_API_KEY="your_api_key"
uv run mewcode
```

```powershell
# Windows PowerShell
$env:OPENAI_API_KEY = "your_api_key"
uv run mewcode
```

#### Anthropic 配置示例

```yaml
providers:
  - name: Anthropic
    protocol: anthropic
    base_url: https://api.anthropic.com
    model: claude-sonnet-4-6
    thinking: true
```

使用 `ANTHROPIC_API_KEY` 提供密钥。

#### OpenAI 兼容服务配置示例

适用于提供 Chat Completions 接口的兼容服务。请将地址和模型名替换为服务商实际值。

```yaml
providers:
  - name: Compatible Provider
    protocol: openai-compat
    base_url: https://your-provider.example/v1
    model: your-model-name
    context_window: 128000  # 可选：显式指定模型上下文窗口
```

> 配置按 `~/.mewcode/config.yaml`、项目 `.mewcode/config.yaml`、项目 `.mewcode/config.local.yaml` 的顺序叠加。最后一项适合保存本机专用的模型或密钥配置，建议加入 `.gitignore`。

### 4. 开始使用

在任意代码仓库目录中运行：

```bash
uv run mewcode
```

向 Agent 描述任务即可，例如：

```text
梳理这个项目的认证流程，并找出可能的空指针风险。
```

如果希望只执行一次任务并将结果输出到标准输出：

```bash
uv run mewcode -p "为当前项目运行测试，并总结失败原因"
```

也可以从命令行指定权限模式，它会覆盖配置文件中的设置：

```bash
uv run mewcode --mode acceptEdits
```

## 使用界面

在对话输入框中：

- 按 `Enter` 发送；按 `Shift+Enter` 或 `Ctrl+J` 换行。
- 输入 `/` 后按 `Tab` 补全斜杠命令。
- 使用 `@path/to/file` 将文件内容带入当前问题。
- 按 `Shift+Tab` 在权限模式之间切换。
- 按 `Ctrl+O` 展开或收起工具调用详情；任务运行期间按 `Esc` 可取消当前响应。

<!-- TODO: Add a GIF or screenshot of a coding task here. Example: ![Editing workflow](docs/images/editing-workflow.gif) -->

## 斜杠命令

| 命令 | 用途 |
| --- | --- |
| `/help [command]` | 查看全部命令或某个命令的详细说明。 |
| `/status` | 查看当前权限模式、会话、上下文使用量、工具数量、记忆与工作目录。 |
| `/plan [task]` | 切换至只读的 Plan 模式；可附带任务描述。 |
| `/compact [focus]` | 手动压缩长对话上下文。 |
| `/clear` | 清除当前对话并创建新会话。 |
| `/session [list\|resume\|new\|delete]` | 列出、恢复、创建或删除会话。 |
| `/memory [list\|clear\|edit]` | 查看、清除或定位自动记忆。 |
| `/rewind [checkpoint] [option]` | 回到文件检查点，可选择恢复代码、对话或两者。 |
| `/permission [mode\|rules\|add\|reset]` | 查看或设置权限模式与规则。 |
| `/mcp` | 查看 MCP Server 连接状态。 |
| `/skill list\|info <name>\|reload` | 管理已发现的 Skills；各 Skill 也可注册成独立斜杠命令。 |
| `/worktree create\|list\|enter\|exit\|status` | 创建、进入、查看、退出或清理 Git worktree。 |
| `/tasks` | 查看、检查或取消后台子 Agent 任务。 |
| `/trace` | 查看 Agent 的父子追踪树和用量汇总。 |

## 权限模式

| 模式 | 读取 | 写入 | 命令 | 适用场景 |
| --- | --- | --- | --- | --- |
| `default` | 自动允许 | 每次询问 | 每次询问 | 日常交互，推荐默认值。 |
| `acceptEdits` | 自动允许 | 自动允许 | 每次询问 | 允许连续代码编辑，但保留 Shell 审批。 |
| `plan` | 自动允许 | 每次询问 | 每次询问 | 先阅读与规划，再决定是否执行。 |
| `bypassPermissions` | 自动允许 | 自动允许 | 自动允许 | 仅用于你完全信任的本地任务。 |
| `custom` | 每次询问 | 每次询问 | 每次询问 | 由规则精确控制。 |
| `dontAsk` | 自动允许 | 自动允许 | 自动允许 | 非交互运行时使用。 |

权限规则可放在下列文件中，并按用户级、项目级、本地级叠加：

```text
~/.mewcode/permissions.yaml
.mewcode/permissions.yaml
.mewcode/permissions.local.yaml
```

例如，在 TUI 中执行下面的命令可将以 `git` 开头的 Shell 命令设为自动允许：

```text
/permission add Bash(git*) allow
```

## 扩展 MewCode

### Skills

Skill 是带 YAML frontmatter 的 `SKILL.md`（或 Markdown）指令包。MewCode 会从以下位置发现它们，优先级从高到低：

```text
.mewcode/skills/       # 当前项目
~/.mewcode/skills/     # 当前用户
mewcode 内置 Skills
```

一个最小的项目级 Skill 可以是：

```markdown
---
name: release-notes
description: 根据当前变更生成发布说明
allowedTools:
  - ReadFile
  - Glob
  - Grep
mode: inline
---

分析当前仓库的变更，并以用户可直接发布的 Markdown 格式生成发布说明。
```

### MCP

在 `.mewcode/config.yaml` 中声明 MCP Server 后，MewCode 会在启动时连接并将其工具注册到 Agent。

```yaml
mcp_servers:
  - name: filesystem
    command: npx
    args:
      - -y
      - "@modelcontextprotocol/server-filesystem"
      - "/path/to/allowed-directory"
```

MCP Server 可以使用 `command` + `args` 的 stdio 方式，也可以使用 `url` 和可选 `headers` 的 HTTP 方式。通过 `/mcp` 检查连接状态。

### Hooks

Hooks 可在启动、会话、轮次、工具调用、压缩等事件上运行命令、插入提示词、发起 HTTP 请求或委派 Agent。下例会在写入前运行检查，并在失败时拒绝这次工具调用：

```yaml
hooks:
  - id: check-before-write
    event: pre_tool_use
    if: tool == "WriteFile"
    reject: true
    action:
      type: command
      command: "your-check-command"
      timeout: 30
```

### 子 Agent、团队与 Worktree

- 内置 Agent 角色包括：`Explore`（只读快速探索）、`Plan`（只读实现规划）、`general-purpose`（通用任务）和可选的 `Verification`（只读验证）。
- 子 Agent 可作为后台任务运行，并通过 `/tasks`、`/trace` 查看状态和关系。
- 团队模式允许多个长期运行的 Agent 通过消息协作；每个成员可在独立 Git worktree 中工作，减少文件冲突。
- `/worktree` 也可用于手动隔离当前会话。若选择删除 worktree，存在未提交改动或新增提交时会要求明确确认。

<!-- TODO: Add an image showing multi-agent / worktree collaboration here. Example: ![Team collaboration](docs/images/team-workflow.png) -->

## 项目结构

```text
mewcode/
├── app.py              # Textual 终端界面与运行时装配
├── agent.py            # Agent 循环、工具执行、上下文压缩与恢复
├── client.py           # Anthropic / OpenAI / 兼容 API 客户端
├── commands/           # 斜杠命令及其处理器
├── tools/              # 文件、Shell、Agent、团队与 Worktree 工具
├── permissions/        # 权限模式、规则、危险命令检测与路径沙箱
├── memory/             # 自动记忆、会话与项目指令
├── skills/             # Skill 发现、解析与执行
├── hooks/              # 生命周期 Hook
├── mcp/                # MCP 客户端与工具包装
├── teams/              # 团队协调、邮箱和执行后端
└── worktree/           # Git worktree 生命周期管理
tests/                  # 单元与集成测试
```

## 开发与测试

安装开发依赖并运行测试：

```bash
uv sync --group dev
uv run pytest tests -q
```

测试覆盖 Agent 主循环、上下文管理、会话与记忆、权限、MCP、Skills、Hooks、子 Agent/团队与 Worktree 等关键模块。

## 当前状态与注意事项
- 外部 MCP Server、Hook 命令和 OpenAI 兼容服务均由你自行配置；请只连接可信来源，并避免把 API Key 提交到仓库。


