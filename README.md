# Personal Skills

[English](README.en.md) | 中文

个人统一的 Codex skill 整理仓库。

## 目录

- `skills/`：确认纳入并整理完成的 skill
- `candidates/`：待筛选的个人 skill 候选与来源记录
- `notes/`：整理决策、改写记录与后续规划
- `catalog.md`：统一索引

## 当前状态

仓库已初始化，候选 skill 先登记、后选择、再复制或重写。当前不修改候选来源文件。

## 主流 Agent 工具与 skill 安装

以下命令按官方文档整理，安装器和包管理器命令可能随版本更新；遇到变化时优先查看对应的官方链接。

### 统一安装本仓库的 skill（推荐）

本仓库采用 `skills/<skill-name>/SKILL.md` 的 Agent Skills 结构。当前在本地 repo 根目录执行：

```bash
# 查看可安装的 skill
npx skills@latest add . --list

# 项目级安装：当前项目可用
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --yes

# 用户级安装：跨项目可用
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --global --yes
```

仓库发布到 GitHub 后，把 `./` 换成 `<owner>/neneko-skill` 即可。`--copy` 可替代默认的 symlink 方式；查看、检查和更新已安装 skill：

```bash
npx skills@latest list
npx skills@latest check
npx skills@latest update
```

统一安装器支持 Codex、Claude Code、OpenCode、Gemini CLI 以及其他兼容 Agent Skills 的工具。详见 [`skills` CLI 文档](https://skills.sh/docs) 和 [vercel-labs/skills](https://github.com/vercel-labs/skills)。

### 安装 Codex CLI（OpenAI）

macOS/Linux：

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex
```

Windows 推荐使用 WSL2；在管理员 PowerShell 中执行 `wsl --install`，进入 WSL 后执行上面的安装命令。首次运行 `codex` 时按提示使用 ChatGPT 登录。

官方文档：[Codex CLI](https://learn.chatgpt.com/docs/codex/cli)、[Windows WSL](https://learn.chatgpt.com/docs/windows/wsl)。

### 安装 Claude Code

原生安装器（推荐）：

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows PowerShell
irm https://claude.ai/install.ps1 | iex

# Windows CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

也可以使用包管理器：

```bash
# Windows
winget install Anthropic.ClaudeCode

# macOS / Linux（Homebrew）
brew install --cask claude-code
```

验证并启动：`claude --version`，然后在项目目录运行 `claude`。原生安装器自动更新；WinGet 和 Homebrew 安装需要手动升级。官方文档：[Claude Code Quickstart](https://code.claude.com/docs/en/quickstart)。

### 安装 OpenCode

最简单的方式：

```bash
curl -fsSL https://opencode.ai/install | bash
```

Node.js/npm 方式：

```bash
npm install -g opencode-ai
opencode
```

Windows 官方推荐 WSL；也可使用 `choco install opencode`、`scoop install opencode` 或 npm。启动后使用 `/connect` 配置模型供应商和 API key。官方文档：[OpenCode](https://opencode.ai/docs)。

### 安装 Gemini CLI（可选）

需要 Node.js 20+。永久安装或临时运行：

```bash
# 全局安装
npm install -g @google/gemini-cli

# 不落盘安装
npx @google/gemini-cli
```

安装后运行 `gemini`，首次使用按提示使用 Google 账号登录。官方文档：[Gemini CLI installation](https://geminicli.com/docs/get-started/installation/)。

## 整理约定

每个纳入的 skill 使用独立目录，并至少包含一个 `SKILL.md`。整理时保留原始能力边界，统一名称、触发条件、输入输出和验证方式；涉及较大改写时，在 `notes/` 留下记录。
