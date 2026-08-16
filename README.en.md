# Personal Skills

[GitHub](https://github.com/XnLemon/Neneko-skill) · English | [中文](README.md)

A personal repository for organizing and maintaining Codex skills.

## Directory layout

- `skills/`: skills that have been selected and standardized
- `candidates/`: personal skill candidates and their source references
- `notes/`: decisions, rewrites, and follow-up plans
- `catalog.md`: the unified skill catalog

## Status

The repository has been initialized. Candidates are cataloged first, then selected, copied, or rewritten. Source files are not modified automatically.

## Mainstream agent and skill installation

The commands below are based on the vendors' current documentation. Installers and package-manager commands can change, so use the linked official documentation when a command no longer works.

### Install a skill from this repository across agents (recommended)

This repository uses the Agent Skills layout: `skills/<skill-name>/SKILL.md`. From the local repository root:

```bash
# List available skills
npx skills@latest add . --list

# Project scope: available in the current project
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --yes

# Global scope: available across projects
npx skills@latest add . --skill handle-review --agent codex --agent claude-code --agent opencode --global --yes
```

Install from GitHub:

```bash
npx skills@latest add XnLemon/Neneko-skill --skill handle-review --agent codex --agent claude-code --agent opencode --yes
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

### Install Codex CLI (OpenAI)

macOS/Linux:

```bash
curl -fsSL https://chatgpt.com/codex/install.sh | sh
codex
```

On Windows, OpenAI recommends WSL2. Run `wsl --install` from an elevated PowerShell, enter WSL, and run the installer above. On first launch, follow the prompt to sign in with ChatGPT.

Official documentation: [Codex CLI](https://learn.chatgpt.com/docs/codex/cli), [Windows WSL](https://learn.chatgpt.com/docs/windows/wsl).

### Install Claude Code

Native installer (recommended):

```bash
# macOS / Linux / WSL
curl -fsSL https://claude.ai/install.sh | bash

# Windows PowerShell
irm https://claude.ai/install.ps1 | iex

# Windows CMD
curl -fsSL https://claude.ai/install.cmd -o install.cmd && install.cmd && del install.cmd
```

Package-manager alternatives:

```bash
# Windows
winget install Anthropic.ClaudeCode

# macOS / Linux (Homebrew)
brew install --cask claude-code
```

Verify with `claude --version`, then run `claude` inside a project directory. Native installs update automatically; WinGet and Homebrew installs require manual upgrades. Official documentation: [Claude Code Quickstart](https://code.claude.com/docs/en/quickstart).

### Install OpenCode

The simplest method:

```bash
curl -fsSL https://opencode.ai/install | bash
```

Node.js/npm alternative:

```bash
npm install -g opencode-ai
opencode
```

On Windows, OpenCode recommends WSL; Chocolatey (`choco install opencode`), Scoop (`scoop install opencode`), and npm are also supported. After starting, use `/connect` to configure an LLM provider and API key. Official documentation: [OpenCode](https://opencode.ai/docs).

### Install Gemini CLI (optional)

Node.js 20+ is required. Install permanently or run without a permanent install:

```bash
# Global installation
npm install -g @google/gemini-cli

# No permanent installation
npx @google/gemini-cli
```

Run `gemini` after installation and follow the first-run Google sign-in flow. Official documentation: [Gemini CLI installation](https://geminicli.com/docs/get-started/installation/).

## Organization conventions

Each included skill gets its own directory and must contain at least one `SKILL.md`. During standardization, preserve the original capability boundaries while making the name, trigger conditions, inputs, outputs, and verification steps consistent. Record substantial rewrites in `notes/`.
