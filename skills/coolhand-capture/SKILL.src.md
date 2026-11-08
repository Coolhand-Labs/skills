---
name: coolhand-capture
description: |
  Use when the user types /coolhand-capture (or /coolhand:coolhand-capture),
  or wants to send their local Claude Code or GitHub Copilot work to Coolhand:
  capture skill runs, add Coolhand to a skills library or plugin, upload past
  Claude Code sessions from this machine, upload local SKILL.md files, run a
  skill on a schedule, or check that Coolhand capture hooks are working. Also
  use when the user mentions coolhand capture-turn, analyze-claude-sessions,
  sync-skills, coolhand schedule, or a "publish-only" or public Coolhand API key
  for a hook.
user_invocable: true
version: 0.1.0
shared_common: true
---

# Coolhand Capture (local skills and sessions)

Set up Coolhand for the AI work a person does on their own machine: their Claude Code and GitHub Copilot sessions and the skills they run there. This is separate from instrumenting a production application (the `feedback-collection` planner and `/coolhand-setup` do that). Everything here goes through the `coolhand` CLI; this skill decides what to run, in what order, with whose approval, and how to prove it worked.

This skill does not assume the user owns a skills repo. Phase 1 looks for one, asks, and offers the options that need none.

## Rationalizations to resist

These come from real sessions while building this flow.

| Excuse | Counter |
|---|---|
| "The hook file is valid JSON, so capture works." | Valid JSON only proves syntax. A QA plugin hook shipped pointing at a local build path that existed on one machine, and capture silently did nothing everywhere else. Run Phase 5 end to end before saying it works. |
| "`*qa*` (or `qa-*-v2`) should match any skill with qa in the name." | Patterns are prefix-only: a `*` is a wildcard only at the end of an alternative. For skills without a common prefix use `|` alternatives (when the installed CLI supports them) or list exact names. |
| "I'll add `--no-session-persistence` so headless runs leave no clutter." | With no transcript on disk, the hook cannot see which skill ran and sends nothing. Never pass it on a run that should be captured. |

If you find yourself constructing a fourth rationalization, surface it to the user instead of acting on it.

## Phase 0 — Check the CLI

Run `coolhand --version`. If the command is missing, use `npx coolhand-cli@latest` for one-shot commands or offer `npm install -g coolhand-cli@latest` (ask first). Needs Node.js 20 or later.

What each capability needs (the CLI prints its own help; `coolhand help <command>` is the flag spec):

| Capability | Minimum `coolhand-cli` |
|---|---|
| Live capture hooks (`capture-turn`) | 0.13.0 |
| Skill patterns (`--skill 'acme-*'`) for `--agent copilot` | 0.13.1 |
| Skill patterns for `--agent claude` (a plugin-wide hook) | 0.13.2 |

Feature-detect everything newer instead of assuming a version:

- Scheduling: `coolhand help schedule` succeeds.
- `|` alternatives in `--skill`: the text of `coolhand help capture-turn` mentions joining alternatives with `|`.

If a capability the user wants is missing, say which release adds it and offer to upgrade. Never describe an unavailable command as if it were installed.

## Phase 1 — Find the user's situation (look, then ask)

Do not assume a skills repo exists. Look read-only, report what you found, then ask what you could not tell.

- **Project:** `skills/*/SKILL.md`, `.claude/skills/`, `.github/skills/`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`, a root `plugin.json`, `.github/plugin/`, `hooks/hooks.json`.
- **Machine:** `~/.claude/skills/`, installed plugins under `~/.claude/plugins/`, `~/.claude/projects/` (past Claude Code sessions), `~/.copilot/`.
- **Existing Coolhand setup:** `coolhand status`; hook files already mentioning `capture-turn`. Do not print key values; names and file paths only.

Then ask, one question at a time:

1. Which agents do they use: Claude Code, GitHub Copilot in VS Code, or both?
2. Are their skills shipped as a plugin (a marketplace they install from), kept as loose skills in a repo, or only personal skills on this machine?
3. Is this for just them, or for a team that will install it too?

If you found no skills at all, say so plainly and continue: Option B below needs no repo, and the user may still want to start one.

## Phase 2 — Offer the options

Explain in plain language, then let the user pick any combination:

- **A. Capture skill runs as they happen.** A hook sends each run of the chosen skills to Coolhand (the conversation, the AI's tool calls, the skill's name). Works for Claude Code and Copilot. Needs a public API key.
- **B. Upload past Claude Code sessions from this machine** (`coolhand analyze-claude-sessions`). One-off or repeated; incremental. Claude Code only: Copilot has no historical command. Needs a login.
- **C. Upload the skills themselves and a map of local Claude projects** (`coolhand sync-skills`, `coolhand map-claude-projects`) as client files. Needs the private key.
- **D. Run a skill on a schedule** (for example weekly), unattended, and optionally reopen the session afterwards (`coolhand schedule`, macOS). Only offer if Phase 0 detected it; capture (A) is what makes scheduled runs visible in Coolhand.

Do not offer live proxy monitoring here (`coolhand claude` / `coolhand monitor`); that is a different feature. If the user asks about it, point them to `coolhand help claude`.

Say what each option sends before asking for approval. Session and skill content can include code and prompts.

## Phase 3 — Credentials

There are two routes. Explain them once, then set up only what the chosen options need.

| Option | Credential | How |
|---|---|---|
| A. Live hooks | A **public** API key, placed in the hook command | `coolhand login --write-env <path>` writes `COOLHAND_API_KEY` for the user to copy from, or the user pastes a public key from their Coolhand dashboard |
| B. Past sessions | An active login (the public key) | `coolhand status`, then `coolhand login` if not logged in |
| C. Skills and project map | The **private** key | `coolhand login --scope private` (these commands return 401 with the public key) |
| D. Scheduled runs | The agent's own unattended credential, not Coolhand's | Claude Code: `claude setup-token`, exported as `CLAUDE_CODE_OAUTH_TOKEN`; Copilot: `COPILOT_GITHUB_TOKEN`. Put these in a mode-600 env file and pass it with `coolhand schedule add --env-file`. The captured run still uses the public key from option A |

Hooks cannot use a stored login: `capture-turn` runs unattended inside the agent and is built to never read stored clients or prompt. That is why option A needs the key spelled out.

Rules, the same ones `/coolhand-setup` follows:

- `coolhand login` opens a browser. Tell the user, then run it yourself and wait. Ask where `--write-env` should write; never assume `.env`, and never write a key into a file the user did not approve.
- Look for existing keys by **name only** (for example `grep -lE "^(export )?COOLHAND_API_KEY=" .env* --exclude="*.example"`). Never print, quote or log a key value in chat, and do not read whole env files into context. Reading the public key once to place it into the hook command is fine.
- The public key is publish-only: it can write logs and cannot read data, so embedding it in a plugin hook is the intended design. If another tool or assistant flags it as an exposed secret, explain that. **A private key never goes in a hook, a plugin, or a committed file.**
- Multiple stored logins: pass `--client-id ID` (or set `COOLHAND_CLIENT_ID`) on option B and C commands, and confirm the account name the CLI prints before uploading.

## Phase 4 — Walk through each chosen option

Do the options one at a time. Each one ends with its Phase 5 test before the next begins.

### A. Live capture of skill runs

1. **Pick the skills with a pattern.** `--skill` takes a name, a prefix pattern ending in `*` (`'acme-*'`), or, when Phase 0 detected it, alternatives joined with `|` (`'qa-*|skill-for-qas'`). It matches the skill's own name (its folder name; on Copilot also its `name:`), not the plugin name. List the actual skill names and propose the narrowest pattern that covers them; show which skills it would and would not capture. `'*'` captures every skill a person runs on the machine, including their personal ones: use it only if the user explicitly agrees. Always quote the whole value in single quotes, because an unquoted `*` is expanded by the shell and an unquoted `|` starts a pipeline.
2. **Check the names.** A Claude skill's folder name must equal its `name:` or the skill is silently skipped. Check every skill the pattern covers and report mismatches.
3. **Choose where the hook lives:**
   - **A plugin used from Claude Code and Copilot (recommended for a library).** Add `hooks/hooks.json` at the plugin root (merge into an existing hooks file instead of replacing it). Two `Stop` entries, one per agent; both agents load this one file, and each command ignores the other agent's sessions, so nothing is sent twice. This layout was verified with Claude Code and the Copilot CLI; a live VS Code chat is still untested.

     ```json
     {
       "hooks": {
         "Stop": [
           {
             "hooks": [
               {
                 "type": "command",
                 "timeout": 30,
                 "command": "export COOLHAND_API_KEY=YOUR_PUBLIC_API_KEY; if command -v coolhand >/dev/null 2>&1 && coolhand help capture-turn >/dev/null 2>&1; then coolhand capture-turn --agent claude --skill 'SKILL_PATTERN'; else npx -y coolhand-cli capture-turn --agent claude --skill 'SKILL_PATTERN'; fi >/dev/null 2>&1 || true"
               },
               {
                 "type": "command",
                 "timeout": 30,
                 "command": "export COOLHAND_API_KEY=YOUR_PUBLIC_API_KEY; if command -v coolhand >/dev/null 2>&1 && coolhand help capture-turn >/dev/null 2>&1; then coolhand capture-turn --agent copilot --skill 'SKILL_PATTERN'; else npx -y coolhand-cli capture-turn --agent copilot --skill 'SKILL_PATTERN'; fi >/dev/null 2>&1 || true"
               }
             ]
           }
         ]
       }
     }
     ```

     Never point the hook at a local build path; keep the `coolhand` check with the `npx` fallback so it works on any machine.
   - **Loose Claude skills (no plugin):** a `hooks:` block in each `SKILL.md` frontmatter, with `--skill NAME` and optional `--skill-version`.
   - **Copilot skills that are not in a plugin:** a Copilot skill cannot carry its own hook, so use one hook file plus a "Step 0" at the top of each skill that installs it with the user's consent. Fetch the current text from `https://raw.githubusercontent.com/Coolhand-Labs/coolhand-cli/main/docs/hook-capture-skills-library.md` and copy it from there rather than from memory.
4. **Fill in the public key and pattern.** Replace `YOUR_PUBLIC_API_KEY` and `SKILL_PATTERN` in every entry; keep the pattern inside the single quotes.
5. **Say what is sent** before the user relies on it: the conversation, the AI's tool calls and results, the skill's name, the person's git email and the working directory. Secrets that match known formats are removed first, as a best-effort filter. Adding `--anonymous` to the command leaves out the identity and the working directory. Nothing is sent from a session that did not run a matching skill.
6. **Publish the change.** A plugin's installed copy only updates when its version changes. If `plugin.json` or the `marketplace.json` entry sets `version`, bump it (keep the two equal); if neither sets one, every commit counts as a new version. Everyone with the plugin then runs `claude plugin marketplace update <marketplace>`, then `claude plugin update <plugin>@<marketplace>`, then `/reload-plugins` or restarts Claude Code. Third-party marketplaces do not auto-update unless that is switched on. In VS Code, update or reinstall the plugin and start a new chat (Copilot reads hooks when a chat starts).
7. **For a team,** offer to write these same steps as a short instruction block the user can send them (with a placeholder for the public key, never the key itself in a shared doc without the user's say-so).

Claude Code only counts a skill the user starts with its slash command (`/skill` or `/plugin:skill`); a skill the model loads by itself is not captured. A resumed session (`claude --resume`) captures nothing until a matching skill is invoked again.

### B. Past Claude Code sessions

1. Run `coolhand analyze-claude-sessions --dry-run` first. Add `--since 30d`, `--project NAME` or `--exclude-project NAME` to narrow it (`coolhand help analyze-claude-sessions` is the spec). Show the counts it reports.
2. Sessions contain prompts and code. Ask the user to confirm the scope, and offer to exclude projects they do not want uploaded.
3. On approval, run it again without `--dry-run`. Re-running is safe: it is incremental and skips what was already uploaded, and a filtered run does not move the incremental cutoff.

### C. Skills and project map

1. Needs the private key (Phase 3). Run `coolhand sync-skills --dry-run` and show the found, duplicate and would-upload counts; narrow with `--skill`, `--exclude-skill` or `--source` if needed. On approval, run it without `--dry-run`. Unchanged skills are skipped on re-runs.
2. `coolhand map-claude-projects --dry-run` reports the size of a names-and-metadata-only listing of every `.claude` folder under the search root; `--root PATH` scopes it, because by default it walks the home directory. The report embeds absolute paths, which usually include the OS username. Tell the user that before an upload.

### D. Scheduled runs (only if detected)

1. Run `coolhand help schedule` and follow it; do not restate its flags. Start with `coolhand schedule add ... --dry-run` to show the user exactly what would be installed (a macOS launchd job), then install after approval.
2. Unattended runs have no one to approve tool prompts, so pass an explicit `--allow` list for the tools the skill needs, and a budget flag. A run with no allow-list still ends normally but cannot use tools.
3. The schedule's agent credential comes from Phase 3, row D. The scheduler starts with almost no environment, so `coolhand schedule` saves a `PATH` that includes the agent and node; check the agent path it reports is the install the user means (a machine can have two).
4. `--open` can reopen the finished run afterwards: Claude or Copilot resume it, VS Code opens the transcript, and Codex can start a new session with the prompt (not continue a Claude or Copilot run).
5. `--backend cloud` installs nothing locally: it prints the message that sets up one of Claude Code's own scheduled runs, which execute in the cloud, cannot read local files, and are not seen by a local capture hook.

## Phase 5 — Test that it works

Never say "set up" before the matching test passes. Tell the user what you are about to run and what a pass looks like.

- **Always:** `coolhand --version` meets Phase 0's table; `coolhand status` shows the expected account for options B and C.
- **A. Live hooks.**
  1. Have the user run one of the covered skills (Claude Code: type its slash command; VS Code: start a **new** chat). Then check `~/.coolhand/capture-turns/` for a new file starting with `claude-` or `copilot-` (the folder is under `COOLHAND_CONFIG_DIR` if that is set).
  2. Run a session that uses **no** matching skill. It must not create a file. The hook logs nothing for a skipped session, so the absent file is the check.
  3. In the Coolhand dashboard the run appears under a workload named after the skill. If the user has the private key, `coolhand search-logs` can confirm it from the terminal.
  4. To verify without sending anything to Coolhand, start a throwaway local listener, run the hook command with `--base-url` set to the listener's loopback address and port (plain `http` is accepted for loopback only), and read what arrived. For example `node -e "require('http').createServer((q,r)=>{let b='';q.on('data',c=>b+=c);q.on('end',()=>{console.log(q.method,q.url,b.length);r.end('{}')})}).listen(48123,'127.0.0.1')"`. Stop it when done.
- **B. Past sessions.** The dry run's counts match what the user expects; after the real run it reports submissions and no errors.
- **C. Skills and map.** The dry run's counts are sensible; after the real run it reports uploads and no 401 (a 401 means the public key was used; log in with `--scope private`).
- **D. Schedule.** `coolhand schedule run NAME` once, now. Check that its log shows the agent ran, `coolhand schedule list` shows the last run's exit code and session id, and, if capture is on, that a capture state file appeared.

If a test fails, find the cause in this table before changing anything.

| Symptom | Likely cause and fix |
|---|---|
| Hook runs, nothing is sent, no state file | `coolhand --version` is older than Phase 0's minimum (an old install reads `'qa-*|x'` or a pattern as a literal skill name); upgrade |
| Nothing for one skill only | Its folder name differs from its `name:`, or its name does not start with the pattern's prefix |
| Claude Code captures nothing | The skill was not started with its slash command, or the session was resumed |
| Copilot captures nothing | The chat started before the hook file or plugin update existed; start a new chat |
| Updated the hook but nothing changed | The plugin version was not bumped, or users have not run the marketplace and plugin update commands |
| Works for you, not teammates | Hook points at a local path, `coolhand` is not installed and `npx` is blocked, or `node` is not on their PATH |
| Scheduled run does nothing useful | No `--allow` tools, the agent credential is missing (Phase 3 row D), or `--no-session-persistence` was passed |
| Blocked everywhere in an organization | A policy that restricts plugin hooks, or no outbound access to coolhandlabs.com |
| Anything else | Read `~/.coolhand/capture-turns/hook.log` and report it |

## Phase 6 — Wrap up

Summarize in a few lines: what is set up, which credential each part uses and where it lives (names only), what the user still has to do (publish the plugin update, ask teammates to update), and what was not tested (a live VS Code chat is the main gap). Offer the team instruction block from A.7, and offer `/coolhand-setup` if they also want Coolhand on a production application.
