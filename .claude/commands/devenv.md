Install a dependency into the running container and bake it into the devcontainer config so it survives rebuilds and is reproducible for teammates.

Usage: `/devenv [type] <args>`

Types: `apt`, `pip`, `npm`, `plugin`, `skill` (optional — inferred if omitted)

Arguments: $ARGUMENTS

---

## Step 1 — Determine the type

Check whether the first word of $ARGUMENTS is a known type keyword (`apt`, `pip`, `npm`, `plugin`, `skill`). 

- **If it is**: that word is the TYPE; the rest is ARGS.
- **If it isn't**: all of $ARGUMENTS is ARGS; infer the TYPE using these rules:
  - Starts with `http://`, `https://`, or looks like `github.com/…` → contains a plugin name before the URL → **plugin**; URL only → **skill**
  - Contains a version specifier (`==`, `~=`, `>=`, `<=`, `!=`, `^`) → **pip**
  - Otherwise → **apt** (system tools like `jq`, `ripgrep`, `shellcheck` are almost always apt)
  - When genuinely ambiguous (e.g. a bare name that could be apt or pip), ask the user before proceeding.

## Step 2 — Execute

### apt — OS package

1. Run `sudo apt-get install -y <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add the package name(s) to the `apt-get install` block in `.devcontainer/postCreate.sh`. If no apt-get block exists yet, add one before the final `echo "==> postCreate complete"` line.
4. If the package is already listed in `postCreate.sh`, say so — no edit needed.

### pip — Python package

1. Run `pip install <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add the package(s) to a `pip install` block in `.devcontainer/postCreate.sh`. Preserve version pins (e.g. `pandas~=2.1`). If no pip block exists yet, add one.
4. If the package is already listed in `postCreate.sh`, say so — no edit needed.

### npm — Node.js global package

1. Run `npm install -g <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add to a `npm install -g` block in `.devcontainer/postCreate.sh`.
4. If the package is already listed in `postCreate.sh`, say so — no edit needed.

### plugin — Claude Code plugin from a Git URL

ARGS format: `<plugin-name> <git-url>` — e.g. `example-skills https://github.com/anthropics/skills.git`

There is no live install step. Writing to `.claude/settings.json` IS the install — Claude Code fetches the plugin on next startup.

1. **Resolve the real marketplace name.** The name is defined in the marketplace repo's manifest, not derivable from the URL:
   - For a GitHub URL: fetch `https://raw.githubusercontent.com/<owner>/<repo>/HEAD/.claude-plugin/marketplace.json` and read the `name` field.
   - If you cannot fetch it, ask the user for the marketplace name before continuing.
   - Example: `https://github.com/anthropics/skills.git` → marketplace name `anthropic-agent-skills` (not `skills`).

2. **Map the URL to a source object:**
   - GitHub URL → `{"source": "github", "repo": "<owner>/<repo>"}`
   - Any other git URL → `{"source": "git", "url": "<full-url>"}`

3. **Merge both keys into `.claude/settings.json`** (do not overwrite other keys):
   ```json
   {
     "extraKnownMarketplaces": {
       "<marketplace-name>": { "source": <source-object> }
     },
     "enabledPlugins": {
       "<plugin-name>@<marketplace-name>": true
     }
   }
   ```
4. If the marketplace and plugin are already in `settings.json`, say so — no edit needed.

### skill — Raw skill from a Git URL

ARGS format: `<git-url>`

There is no live install step. Raw skills are cloned into `~/.claude/skills/` on container rebuild via `postCreate.sh`.

1. Add the URL to the `CLAUDE_SKILL_REPOS` array in `.devcontainer/postCreate.sh`.
2. If it's already listed, say so — no edit needed.
3. Tell the user to rebuild the container to activate the skill.

---

## After every successful operation

Report three things concisely:
- What was installed (or configured) and where
- What was changed in which config file (show only the diff, not the full file); or confirm it was already present
- What a fresh teammate needs to do: rebuild the container (apt/pip/npm/skill) or restart Claude Code (plugin)
