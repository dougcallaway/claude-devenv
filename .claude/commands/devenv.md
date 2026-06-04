Install a dependency into the running container and bake it into the devcontainer config so it survives rebuilds and is reproducible for teammates.

Usage: `/devenv <type> <args>`

Types: `apt`, `pip`, `npm`, `plugin`, `skill`

Arguments: $ARGUMENTS

---

Parse the first word of the arguments as TYPE and the remainder as ARGS. Then follow the instructions for that type exactly.

## apt — OS package

1. Run `sudo apt-get install -y <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add the package name(s) to the `apt-get install` block in `.devcontainer/postCreate.sh`. If no apt-get install block exists yet, add one before the final `echo "==> postCreate complete"` line.
4. Show only the added line(s), not the full file.

## pip — Python package

1. Run `pip install <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add the package(s) to a `pip install` block in `.devcontainer/postCreate.sh`. Preserve any version pins the user provided (e.g. `pandas~=2.1`). If no pip install block exists yet, add one.
4. Show only the added line(s), not the full file.

## npm — Node.js global package

1. Run `npm install -g <ARGS>` in the shell.
2. If it fails, stop and report the error. Do not edit any files.
3. If it succeeds, add to a `npm install -g` block in `.devcontainer/postCreate.sh`.
4. Show only the added line(s), not the full file.

## plugin — Claude Code plugin from a Git URL

ARGS format: `<plugin-name> <git-url>` — e.g. `example-skills https://github.com/anthropics/skills.git`

There is no live install step for plugins. Writing to `.claude/settings.json` IS the install — Claude Code fetches the plugin on next startup.

1. **Resolve the real marketplace name.** The name is defined in the marketplace repo's manifest, not derivable from the URL. To find it:
   - For a GitHub URL: fetch `https://raw.githubusercontent.com/<owner>/<repo>/HEAD/.claude-plugin/marketplace.json` (or `marketplace.json` at root) and read the `name` field.
   - If you cannot fetch it, ask the user for the marketplace name before continuing.
   - Example: `https://github.com/anthropics/skills.git` → marketplace name `anthropic-agent-skills` (not `skills`).

2. **Map the URL to a source object:**
   - GitHub URL (`github.com/<owner>/<repo>`) → `{"source": "github", "repo": "<owner>/<repo>"}`
   - Any other git URL → `{"source": "git", "url": "<full-url>"}`

3. **Merge both keys into `.claude/settings.json`** (do not overwrite other keys):
   ```json
   {
     "extraKnownMarketplaces": {
       "<marketplace-name>": {
         "source": <source-object>
       }
     },
     "enabledPlugins": {
       "<plugin-name>@<marketplace-name>": true
     }
   }
   ```

4. Tell the user: the settings are written; the plugin will be fetched automatically on the next Claude Code session start (or after a `/reload`).

## skill — Raw skill from a Git URL

ARGS format: `<git-url>`

There is no live install step. Raw skills must be cloned into `~/.claude/skills/`, which happens on container rebuild via `postCreate.sh`.

1. Add the URL to the `CLAUDE_SKILL_REPOS` array in `.devcontainer/postCreate.sh`.
2. Show only the added line.
3. Tell the user to rebuild the container to activate the skill.

---

## After every successful operation

Report three things concisely:
- What was installed (or configured) and where
- What was changed in which config file (show the diff, not the full file)
- What a fresh teammate needs to do: rebuild the container (for apt/pip/npm/skill) or restart Claude Code (for plugin)
