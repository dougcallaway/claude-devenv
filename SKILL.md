---
name: claude-devenv
description: Scaffold and manage sandboxed, Claude-enabled development environments using VS Code devcontainers. Use this skill whenever the user wants to create a new devcontainer for a project that includes Claude Code, set up Claude API access in a container, configure a reproducible AI-assisted dev environment, add language runtimes or tools to an existing devcontainer, or understand how to wire Claude Code into a container-based workflow. Trigger on phrases like "devcontainer", "sandbox", "Claude-enabled environment", "set up Claude in a container", "add Python/Node to my devcontainer", or "I want a reproducible dev environment with Claude".
---

# Claude-Enabled Devcontainer Skill

This skill covers the full lifecycle of sandboxed Claude-enabled development environments: scaffolding from scratch, wiring in Claude Code + API access, and evolving existing environments.

The reference example lives at `/workspaces/wiki/.devcontainer/` — a minimal but production-ready setup. Read those files if you need a concrete anchor.

---

## Core Anatomy

Every Claude-enabled devcontainer needs three files:

```
.devcontainer/
├── devcontainer.json   ← container spec + features + volume mounts
└── postCreate.sh       ← runs once after the container builds
```

Optionally, a `Dockerfile` alongside these if you need to build a custom image.

---

## Key Decisions to Make With the User

Before generating any files, resolve these:

1. **Base image** — `mcr.microsoft.com/devcontainers/base:noble` (Ubuntu 24.04) is the safe default. Use `debian`, `alpine`, or a language-specific image if there's a reason.

2. **Language/runtime stack** — Which runtimes does the project need? See [language-stacks.md](references/language-stacks.md) for feature snippets and package lists.

3. **API key strategy** — How should `ANTHROPIC_API_KEY` reach the container? Three options:
   - **Host passthrough** (recommended for local dev): `"remoteEnv": {"ANTHROPIC_API_KEY": "${localEnv:ANTHROPIC_API_KEY}"}` in devcontainer.json
   - **Manual post-start**: user runs `claude` and authenticates interactively after first start
   - **Codespaces / CI secret**: use `"containerEnv"` referencing the secret name

4. **Persistent state** — The `.claude` config directory stores auth, settings, and memory. Always mount it as a named volume so it survives container rebuilds. See the volume pattern below.

---

## Scaffolding a New Environment

### devcontainer.json

```jsonc
{
  "name": "<project-name>",
  "image": "mcr.microsoft.com/devcontainers/base:noble",
  "features": {
    "ghcr.io/anthropics/devcontainer-features/claude-code:1.0": {},
    // add language features here — see language-stacks.md
  },
  "customizations": {
    "vscode": {
      "extensions": ["anthropic.claude-code"]
    }
  },
  "remoteUser": "vscode",
  "containerUser": "vscode",
  "mounts": [
    // Named volume — persists .claude config (auth, settings, memory) across rebuilds
    "source=claude-code-config-${devcontainerId},target=/home/vscode/.claude,type=volume"
  ],
  "remoteEnv": {
    "ANTHROPIC_API_KEY": "${localEnv:ANTHROPIC_API_KEY}"
  },
  "postCreateCommand": "bash .devcontainer/postCreate.sh"
}
```

**Why the named volume?** The `devcontainerId` suffix scopes the volume to this project, so multiple devcontainer projects don't share state. On rebuild, the volume persists — Claude Code's auth, memory, and settings survive.

**Why both `remoteUser` and `containerUser`?** `containerUser` sets the user that runs the container process; `remoteUser` sets the user VS Code connects as. Keeping them in sync avoids permission surprises.

### postCreate.sh

```bash
#!/usr/bin/env bash
set -euo pipefail

# -- version pins -- update these lines to upgrade dependencies
# PANDAS_VERSION="2.1"
# HTTPX_VERSION="0.27"

echo "==> [1/3] Fixing volume ownership"
sudo chown -R vscode:vscode /home/vscode/.claude

echo "==> [2/3] Installing system tools"
sudo apt-get update -qq
sudo apt-get install -y -qq <packages>

echo "==> [3/3] Installing Python packages"
# pip3 install --break-system-packages \
#   "pandas~=${PANDAS_VERSION}" \
#   "httpx~=${HTTPX_VERSION}"

echo "==> postCreate complete"
```

Key conventions:
- `set -euo pipefail` — fail fast on any error; don't silently swallow problems
- Volume ownership fix **must come first** — the named volume mounts before postCreate runs, but may be owned by root if freshly created
- **Version pins at the top as variables** — `~=` (Python) locks MAJOR.MINOR and allows patch updates; `^` (npm) allows minor+patch within a major. Upgrading a dependency is then a one-line diff at the top of the file, not a search through install commands
- Populate the version-pin block and install commands from the user's requested packages; never leave placeholder comments in the final output

---

## Evolving an Existing Environment

### Adding a language runtime

Add the devcontainer feature to `devcontainer.json` → rebuild the container. See [language-stacks.md](references/language-stacks.md) for the exact feature URIs and common package lists.

### The live-to-bake loop for packages

The natural workflow for adding a package mid-session:

1. **Install live** — unblocks you immediately, lets you verify the package actually works for the task
2. **Bake it in** — once it works, add the exact working command to `postCreate.sh` before moving on
3. **Rebuild to confirm** — the next rebuild proves postCreate.sh is correct

The failure mode is skipping step 2: the package works for you today, but the next person to rebuild (or you, two weeks later) gets a broken environment with no explanation. Treat live-installed packages as temporary until they're in postCreate.sh.

### Tracking dependencies with a hook

The dependency-tracking hook is bundled with this skill at [scripts/detect-dep-install.sh](scripts/detect-dep-install.sh). Include it automatically when scaffolding — no manual setup needed.

As part of the scaffolding output, write two additional files:

**`.claude/hooks/detect-dep-install.sh`** — copy the bundled script verbatim.

**`.claude/settings.json`** — create or merge:
```json
{
  "hooks": {
    "PostToolUse": [{
      "matcher": "Bash",
      "hooks": [{"type": "command", "command": "bash .claude/hooks/detect-dep-install.sh"}]
    }]
  }
}
```

If `.claude/settings.json` already exists, merge the hook entry into the existing `PostToolUse` array rather than overwriting.

When the hook fires, assess whether the dependency is worth baking in:
- **Bake it** — if it's a library the project will use repeatedly, or any team member would need
- **Skip it** — if it's a one-off diagnostic tool (e.g., `htop`, `strace`) or a quick experiment that didn't pan out

When suggesting an update to `postCreate.sh`, show only the line to add — not the full file.

### Updating pinned versions

Update the version variables at the top of `postCreate.sh`. Use `~=` (Python) or `^` (npm) to allow patch-level updates while locking the minor.

### When to rebuild vs. install live

| Situation | Action |
|-----------|--------|
| Adding a new language runtime | Rebuild (feature must be in devcontainer.json) |
| Adding a Python/npm package | Live install to test, then bake into postCreate.sh, rebuild to confirm |
| Updating a package version | Update postCreate.sh, rebuild |
| Changing volume mounts | Rebuild (mounts are set at container creation) |
| Tweaking VS Code extensions | Rebuild or install from Extensions panel — both work |

---

## Adapting to Non-Devcontainer Runtimes

If the user is on Docker Compose, Gitpod, or a raw Dockerfile:

- **Docker Compose**: move features into a `Dockerfile` using the devcontainer feature install script (`devcontainer features install`), or install Claude Code via `npm install -g @anthropic-ai/claude-code`
- **Gitpod**: use `.gitpod.yml` + `Dockerfile`; install Claude Code in the Dockerfile; pass `ANTHROPIC_API_KEY` via Gitpod environment variables UI
- **Raw Dockerfile**: `RUN npm install -g @anthropic-ai/claude-code`; mount a host directory or use a Docker volume for `.claude`

In all cases, the persistent-volume-for-.claude pattern applies regardless of runtime.

---

## Output

When scaffolding, produce all four files and show them in-context before writing:
1. `.devcontainer/devcontainer.json`
2. `.devcontainer/postCreate.sh`
3. `.claude/hooks/detect-dep-install.sh` (copied from bundled script)
4. `.claude/settings.json` (hook registration)

When evolving, show only the diff (changed lines in context). Always explain *why* each non-obvious config choice was made.

Commit `.claude/` alongside `.devcontainer/` — both are project dev environment config. Nothing sensitive belongs in `.claude/settings.json`; API keys go in `remoteEnv` in `devcontainer.json`. Teammates who want personal Claude preferences can layer them via `~/.claude/settings.json`, which merges with the project-level file.

Add `.claude/settings.local.json` to `.gitignore` — Claude Code writes per-user permission grants there, which are session-specific and meaningless to other users.

After writing files, tell the user what to do next:
1. Open in VS Code → "Reopen in Container" (or rebuild if already in container)
2. Set `ANTHROPIC_API_KEY` in host shell if using host passthrough, or authenticate via `claude` CLI after first start
3. Verify with `claude --version` in the container terminal
