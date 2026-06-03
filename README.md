# claude-devenv

A Claude Code skill that scaffolds and manages Claude-enabled VS Code devcontainers.

Invoke it by describing what you want — it asks a few questions and writes the devcontainer files for you:

> "Set up a Claude-enabled devcontainer for this Python project"
> "Add Node.js to my devcontainer"
> "I want a reproducible dev environment with Claude and Postgres"

---

## Getting Started

### 1. Prerequisites

Install these before using the skill:

| Prerequisite | Notes |
|---|---|
| [VS Code](https://code.visualstudio.com/) | With the [Dev Containers extension](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-containers) |
| Docker | [Docker Desktop](https://www.docker.com/products/docker-desktop/) on macOS/Windows; [Docker Engine](https://docs.docker.com/engine/install/) on Linux |
| Git | Any recent version |
| Claude Code | `npm install -g @anthropic-ai/claude-code` |

### 2. Install the skill

**macOS / Linux / WSL:**
```bash
git clone --depth 1 https://github.com/dougcallaway/claude-devenv \
  ~/.claude/skills/claude-devenv
```

**Windows (PowerShell):**
```powershell
git clone --depth 1 https://github.com/dougcallaway/claude-devenv `
  "$env:USERPROFILE\.claude\skills\claude-devenv"
```

No further configuration needed — Claude Code auto-discovers skills in `~/.claude/skills/`.

### 3. Open or create your project

Open an existing project in VS Code, or create/clone one first:

```bash
# New empty project
mkdir my-project && cd my-project && code .

# From a remote repo
git clone <url> && cd <project-name> && code .
```

### 4. Ask Claude to scaffold the devcontainer

In the Claude Code panel, describe what you want. For example:

> "Set up a Claude-enabled devcontainer for this project"

Claude will ask a few questions (base image, language stack, auth method, etc.) and write the devcontainer files.

### 5. Reopen in the container

VS Code will prompt **"Reopen in Container"** after the files are written — click it. On first open, the container image builds and `postCreate.sh` runs.

### 6. Authenticate (manual step)

**OAuth (personal subscription):** The Claude Code extension prompts you to sign in automatically when the container opens. Complete the browser login — this is easier than terminal auth because the extension runs on the host and can open a browser natively.

**API key:** Ensure `ANTHROPIC_API_KEY` is set in your host environment *before* opening the container. Where to set it depends on your OS — ask Claude or see [references/environments.md](references/environments.md).

### 7. Verify

```bash
claude --version
```

---

## Keeping the skill up to date

```bash
# macOS / Linux / WSL
git -C ~/.claude/skills/claude-devenv pull --ff-only

# Windows (PowerShell)
git -C "$env:USERPROFILE\.claude\skills\claude-devenv" pull --ff-only
```

---

## What the skill generates

| File | Purpose |
|---|---|
| `.devcontainer/devcontainer.json` | Container spec: base image, features, volume mounts, env vars |
| `.devcontainer/postCreate.sh` | Runs once after build: installs packages, clones skills |
| `.claude/hooks/detect-dep-install.sh` | Reminds you to bake live-installed packages into `postCreate.sh` |
| `.claude/settings.json` | Registers the hook; can also declare marketplace skill sources |

Commit all of these alongside your source code. Add `.claude/settings.local.json` to `.gitignore` — Claude Code writes per-user permission grants there.
