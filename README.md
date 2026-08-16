# Personal Skills

[GitHub](https://github.com/XnLemon/Neneko-skill) · [English](README.en.md) | 中文

个人统一的 Codex skill 整理仓库。

## 目录

- `skills/`：确认纳入并整理完成的 skill
- `candidates/`：待筛选的个人 skill 候选与来源记录
- `notes/`：整理决策、改写记录与后续规划
- `catalog.md`：统一索引

## 当前状态

仓库已初始化，候选 skill 先登记、后选择、再复制或重写。当前已纳入
`handle-review` 和 `neneko-cr`；候选来源文件仍不直接修改。

## 日常开发工作流

本仓库当前纳入的两个 skill 都属于日常开发工作流：

- `handle-review`：面向单个 PR review thread，核对评论是否适用于当前 HEAD，判断问题有效性和范围，处理最小完整修复、回归测试、验证和中英文回复。
- `neneko-cr`：面向任意项目的完整 CR 流程，负责 PR 分流、CI 和评审门禁、证据链评论、自有 PR 维护、审批/合并决策以及审计报告。

简单区分：`handle-review` 更偏单条评论和局部处理，`neneko-cr` 更偏仓库级批量审查和运行编排；两者可以独立使用，也可以配合使用。

## 跨 Agent 安装 skill

本仓库采用 `skills/<skill-name>/SKILL.md` 的 Agent Skills 结构。当前在本地 repo 根目录执行：

```bash
# 查看可安装的 skill
npx skills@latest add . --list

# 项目级安装：当前项目可用
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --yes

# 安装通用 CR skill
npx skills@latest add . --skill neneko-cr --agent codex --agent claude-code --agent opencode --yes

# 用户级安装：跨项目可用
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --global --yes

# 用户级安装通用 CR skill
npx skills@latest add . --skill neneko-cr --agent codex --agent claude-code --agent opencode --global --yes
```

从 GitHub 安装：

```bash
npx skills@latest add XnLemon/Neneko-skill --skill handle-review --agent codex --agent claude-code --agent opencode --yes
npx skills@latest add XnLemon/Neneko-skill --skill neneko-cr --agent codex --agent claude-code --agent opencode --yes
```

如需本地开发或同步仓库：

```bash
git clone https://github.com/XnLemon/Neneko-skill.git
cd Neneko-skill
```

`--copy` 可替代默认的 symlink 方式；查看、检查和更新已安装 skill：

```bash
npx skills@latest list
npx skills@latest check
npx skills@latest update
```

统一安装器支持 Codex、Claude Code、OpenCode、Gemini CLI 以及其他兼容 Agent Skills 的工具。详见 [`skills` CLI 文档](https://skills.sh/docs) 和 [vercel-labs/skills](https://github.com/vercel-labs/skills)。

## 整理约定

每个纳入的 skill 使用独立目录，并至少包含一个 `SKILL.md`。整理时保留原始能力边界，统一名称、触发条件、输入输出和验证方式；涉及较大改写时，在 `notes/` 留下记录。
