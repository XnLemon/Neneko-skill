# Personal Skills

[GitHub](https://github.com/XnLemon/Neneko-skill) · English | [中文](README.md)

A personal repository for organizing and maintaining Codex skills.

## Directory layout

- `skills/`: skills that have been selected and standardized
- `candidates/`: personal skill candidates and their source references
- `notes/`: decisions, rewrites, and follow-up plans
- `catalog.md`: the unified skill catalog

## Status

The repository has been initialized. Candidates are cataloged first, then selected, copied, or rewritten. `handle-review` and `neneko-cr` are now included. Source files are not modified automatically.

## Daily Development Workflows

The two included skills are both part of the daily development workflow:

- `handle-review`: Handles an individual PR review thread by verifying the claim against current HEAD, judging validity and scope, applying the smallest complete fix, adding regression coverage, validating it, and drafting English-first replies with folded Chinese.
- `neneko-cr`: Runs a repository-agnostic CR workflow for triage, CI and review gates, evidence-backed comments, own-PR maintenance, approval/merge decisions, and auditable reports.

In short, `handle-review` focuses on one comment or thread, while `neneko-cr` focuses on repository-level review orchestration. They can be used independently or together.

## Install skills across agents

This repository uses the Agent Skills layout: `skills/<skill-name>/SKILL.md`. From the local repository root:

```bash
# List available skills
npx skills@latest add . --list

# Project scope: available in the current project
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --yes

# Install the general-purpose CR skill
npx skills@latest add . --skill neneko-cr --agent codex --agent claude-code --agent opencode --yes

# Global scope: available across projects
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --global --yes

# Global install for the general-purpose CR skill
npx skills@latest add . --skill neneko-cr --agent codex --agent claude-code --agent opencode --global --yes
```

Install from GitHub:

```bash
npx skills@latest add XnLemon/Neneko-skill --skill handle-review --agent codex --agent claude-code --agent opencode --yes
npx skills@latest add XnLemon/Neneko-skill --skill neneko-cr --agent codex --agent claude-code --agent opencode --yes
```

For local development or synchronization:

```bash
git clone https://github.com/XnLemon/Neneko-skill.git
cd Neneko-skill
```

Use `--copy` instead of the default symlink method when symlinks are not suitable. To list, check, or update installed skills:

```bash
npx skills@latest list
npx skills@latest check
npx skills@latest update
```

The installer supports Codex, Claude Code, OpenCode, Gemini CLI, and other tools compatible with the Agent Skills ecosystem. See the [`skills` CLI documentation](https://skills.sh/docs) and [vercel-labs/skills](https://github.com/vercel-labs/skills).

## Organization conventions

Each included skill gets its own directory and must contain at least one `SKILL.md`. During standardization, preserve the original capability boundaries while making the name, trigger conditions, inputs, outputs, and verification steps consistent. Record substantial rewrites in `notes/`.
