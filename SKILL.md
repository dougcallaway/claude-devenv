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

3. **Authentication method** — Ask whether use is interactive/personal or automated/shared, then choose:

   | Scenario | Method | Setup |
   |----------|--------|-------|
   | Personal daily dev (subscription) | OAuth login | Nothing in `devcontainer.json`; user runs `claude` after first start and logs in via browser. Token stored in `~/.claude/`, persists via named volume. |
   | Team or shared environment | API key | Add `"remoteEnv": {"ANTHROPIC_API_KEY": "${localEnv:ANTHROPIC_API_KEY}"}`. Where the key must be set on the host varies by environment — see [environments.md](references/environments.md). |
   | CI/CD or automated/agentic workflows | API key | Same as above, or `"containerEnv"` referencing a CI secret. OAuth tokens aren't designed for unattended use. |

   **Cost tradeoff:** OAuth subscription is fixed monthly cost ($20/mo Pro, $100/mo Max) — better for heavy interactive use. API key is pay-per-token — better for variable or automated use, but token costs accumulate quickly in agentic contexts (long context, many tool calls).

4. **Persistent state** — The `.claude` config directory stores auth tokens (OAuth or API key), settings, and memory. Always mount it as a named volume so it survives container rebuilds — this applies equally to OAuth and API key auth. Note: on Codespaces, named volumes are lost when the Codespace is deleted — warn the user they'll need to re-authenticate regardless of auth method. See [environments.md](references/environments.md) for details.

5. **Claude skills** — Should any team-shared or third-party Claude skills be pre-installed for every developer? Collect the list and choose an install method for each:
   - **Git URL** → add to `CLAUDE_SKILL_REPOS` in `postCreate.sh`; cloned into `~/.claude/skills/` on build
   - **Marketplace plugin** → add to `CLAUDE_SKILL_MARKETPLACES` in `postCreate.sh`; also register the marketplace in `.claude/settings.json` if it isn't the default Anthropic one
   - **Local path (skill lives in the repo or a sibling directory)** → bind-mount it into the container; see _Adding Claude skills_ under Evolving

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

# -- Claude skills: git repos cloned into ~/.claude/skills/ --
CLAUDE_SKILL_REPOS=(
  # "https://github.com/org/skill-name"
)

# -- Claude skills: marketplace plugins --
# Non-default marketplaces must be registered in .claude/settings.json (extraKnownMarketplaces)
CLAUDE_SKILL_MARKETPLACES=(
  # "plugin-name@marketplace-name"
)

echo "==> [1/N] Fixing volume ownership"
sudo chown -R vscode:vscode /home/vscode/.claude

echo "==> [2/N] Installing Claude skill repos"
if (( ${#CLAUDE_SKILL_REPOS[@]} > 0 )); then
  mkdir -p /home/vscode/.claude/skills
  for url in "${CLAUDE_SKILL_REPOS[@]}"; do
    skill_name=$(basename "$url" .git)
    target="/home/vscode/.claude/skills/${skill_name}"
    if [ -d "$target" ]; then
      git -C "$target" pull --ff-only
    else
      git clone --depth 1 "$url" "$target"
    fi
  done
fi

echo "==> [3/N] Installing Claude marketplace skills"
for plugin in "${CLAUDE_SKILL_MARKETPLACES[@]+"${CLAUDE_SKILL_MARKETPLACES[@]}"}"; do
  claude plugin install "$plugin" --scope user
done

echo "==> [4/N] Installing system tools"
sudo apt-get update -qq
sudo apt-get install -y -qq <packages>

echo "==> [5/N] Installing Python packages"
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

### Adding Claude skills

Three install paths depending on where the skill comes from:

**Git URL** — add to `CLAUDE_SKILL_REPOS` in `postCreate.sh`. On every rebuild, the loop clones fresh repos and pulls existing ones (`--ff-only` keeps it safe). Skills land in `~/.claude/skills/` inside the named volume and persist across rebuilds.

**Marketplace plugin** — add to `CLAUDE_SKILL_MARKETPLACES` in `postCreate.sh`. For the default Anthropic marketplace nothing else is needed. For a third-party or team marketplace, also add an `extraKnownMarketplaces` entry to `.claude/settings.json` (committed to the repo so all developers get it):

```json
{
  "extraKnownMarketplaces": {
    "my-team-skills": {
      "source": { "source": "github", "repo": "my-org/claude-skills" }
    }
  }
}
```

Then reference it in `postCreate.sh` as `"skill-name@my-team-skills"`.

**Local path (skill lives in the repo or a sibling directory)** — bind-mount the skill directory into the container at the path Claude Code expects. Add a `mounts` entry to `devcontainer.json` alongside the existing named-volume mounts:

```jsonc
"mounts": [
  // existing named volumes
  "source=claude-code-config-${devcontainerId},target=/home/vscode/.claude,type=volume",
  // bind-mount a skill from within the repo
  "source=${localWorkspaceFolder}/tools/my-skill,target=/home/vscode/.claude/skills/my-skill,type=bind,readonly",
  // bind-mount a skill from a sibling directory
  "source=${localWorkspaceFolder}/../my-skill,target=/home/vscode/.claude/skills/my-skill,type=bind,readonly"
]
```

`${localWorkspaceFolder}` is resolved by the devcontainer runtime to the host path of the repo root. Use `readonly` — the container should read skills, not modify them. The bind mount at the more-specific path (`~/.claude/skills/my-skill`) overlays the named volume at that subdirectory, so the rest of `~/.claude` is unaffected.

Note: bind-mounted skills are not cloned or updated by `postCreate.sh` — the host directory is the source of truth. Rebuilding the container picks up whatever is currently on disk on the host.

### Updating pinned versions

Update the version variables at the top of `postCreate.sh`. Use `~=` (Python) or `^` (npm) to allow patch-level updates while locking the minor.

### When to rebuild vs. install live

| Situation | Action |
|-----------|--------|
| Adding a new language runtime | Rebuild (feature must be in devcontainer.json) |
| Adding a Python/npm package | Live install to test, then bake into postCreate.sh, rebuild to confirm |
| Updating a package version | Update postCreate.sh, rebuild |
| Adding a skill (git URL or marketplace) | Add to postCreate.sh arrays, rebuild |
| Adding a skill (local path) | Add bind mount to devcontainer.json, rebuild |
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
2. **OAuth auth**: the Claude Code VS Code extension will prompt you to sign in automatically — complete the browser login from there. This is easier than terminal auth in a devcontainer because the extension runs on the host side and can open a browser natively. Fallback: run `claude` in the container terminal (may require port forwarding or a display to complete the browser step).
   **API key auth**: ensure `ANTHROPIC_API_KEY` is set on the host (see [environments.md](references/environments.md) for where to set it per OS)
3. Verify with `claude --version` in the container terminal
