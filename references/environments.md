# Environment-Specific Notes

Consult this when the user's host environment is known. The main variables
are: how `ANTHROPIC_API_KEY` reaches the container (API key auth only — OAuth
users authenticate interactively inside the container and need no host env var),
mount performance characteristics, and whether named volumes survive environment
teardown.

---

## WSL (Windows Subsystem for Linux)

**API key** — Must be set in the WSL shell profile (`.bashrc` or `.zshrc`),
not in Windows System Properties → Environment Variables. Windows env vars
don't propagate into WSL unless explicitly bridged. The user should add:
```bash
export ANTHROPIC_API_KEY="sk-ant-..."
```
to their WSL `~/.bashrc` or `~/.zshrc`, then `source` it or open a new terminal.

**Docker** — Docker Desktop for Windows with WSL2 integration enabled (Docker
Desktop → Settings → Resources → WSL Integration).

**Mounts** — Avoid bind mounts that cross the WSL↔Windows filesystem boundary
(paths under `/mnt/c/...`). Cross-filesystem I/O is slow. Named volumes and
bind mounts within the WSL filesystem (`/home/...`, `/workspaces/...`) are
full speed.

---

## macOS

**API key** — VS Code on macOS spawns a login shell to read env vars, which
loads `.zprofile` but NOT `.zshrc` (interactive-only). If the user set the
key in `.zshrc` and it isn't passing through, move it to `~/.zprofile`.
Fish shell: `~/.config/fish/config.fish` works for both login and interactive.

**Docker** — Docker Desktop required (or OrbStack, which is lighter and faster).

**Mounts** — Bind mounts to the macOS filesystem go through a translation
layer (VirtioFS in recent Docker Desktop, older installs used osxfs). Named
volumes stay inside the Docker VM and are significantly faster for I/O-heavy
workloads. Prefer named volumes for anything the container writes frequently
(`.claude`, build caches, etc.).

**Apple Silicon (M1/M2/M3/M4)** — Most devcontainer base images are
multi-arch (amd64 + arm64). If a package with native extensions fails to
install, try adding `"runArgs": ["--platform=linux/amd64"]` to
`devcontainer.json` as a fallback, accepting the Rosetta emulation overhead.

---

## GitHub Codespaces

**API key** — Set as a Codespaces secret, not a shell profile. GitHub →
Settings → Codespaces → Secrets (or repo-level: repo → Settings → Secrets →
Codespaces). Codespaces injects secrets as env vars on the VM, so
`${localEnv:ANTHROPIC_API_KEY}` in `remoteEnv` works as-is — the user just
needs the secret configured before launching the Codespace.

**Volume persistence** — Named volumes scoped to `${devcontainerId}` persist
within a Codespace instance but are **deleted when the Codespace is deleted**.
This means `.claude` auth and memory are wiped on Codespace deletion. Warn
the user: they'll need to re-authenticate with `claude` after creating a new
Codespace. (There is no workaround that avoids re-auth on full Codespace
deletion.)

**Prebuilds** — For projects with heavy `postCreate.sh` scripts, suggest
enabling Codespaces prebuilds (repo → Settings → Codespaces → Prebuilds).
Prebuilds cache the container image so new Codespaces start in seconds.
`postCreateCommand` still runs, but image layers are pre-built.

**No Docker Desktop needed** — Containers run natively in the Codespace VM.
Port forwarding is handled by GitHub's infrastructure; ports can be made
public or private from the Ports panel.

---

## Linux (native Docker Engine)

**API key** — Works normally from shell profile (`.bashrc`/`.zshrc`).

**Docker** — Docker Engine, not Docker Desktop. If the user gets a permission
error running Docker, they need to add themselves to the `docker` group:
```bash
sudo usermod -aG docker $USER
# then log out and back in (or: newgrp docker)
```

**Mounts** — No translation layer; bind mounts and named volumes are both
full speed. No special considerations.

---

## Volume naming across environments

The `${devcontainerId}` variable in mount source names is resolved by the
devcontainer CLI and is consistent across VS Code, Codespaces, and the
`devcontainer` CLI. It's derived from a hash of the workspace folder path —
if the project is moved to a different path, the volume name changes and the
container starts with a fresh volume (re-auth required for `.claude`).

This is usually fine. If volume continuity across path changes matters, use
an explicit static name (e.g., `source=myproject-claude-config`) instead of
the `${devcontainerId}` pattern — but be aware that this breaks isolation if
the same project is opened from multiple paths simultaneously.
