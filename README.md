# Coolhand Skills

Skills for AI/LLM product teams from [Coolhand](https://coolhandlabs.com), shipped as one Claude Code plugin. The flagship, `feedback-collection`, finds AI/LLM workflows in your codebase and implements best-practice human feedback collection — either against the [managed Coolhand backend](https://coolhandlabs.com) or scaffolded onto your own servers.

## What it does

Run `/feedback-collection` in any project and the planner skill will:

1. **Scan** your codebase (or a scoped path) for LLM inference calls — OpenAI, Anthropic, LangChain, LlamaIndex, Ollama, Bedrock, and more
2. **Identify** feedback collection opportunities for each workflow, preferring passive signals (zero UI changes) over active ones
3. **Design** a strategy covering matching, signal quality, and user identity — with privacy-safe `creator_unique_id` handling
4. **Propose** a concrete plan for your approval before touching any code
5. **Dispatch** to one of two implementation skills:
   - `coolhand-integration` — installs the Coolhand SDKs against `coolhandlabs.com` (default; fastest setup; includes the analytics dashboard)
   - `self-hosted-feedback` — scaffolds Coolhand-compatible endpoints on your own backend (for privacy/compliance/data-residency requirements)

`/coolhand-setup` (`/coolhand:coolhand-setup` in the plugin) is the setup walkthrough. Run it first if you are not sure what you need: it asks whether you are setting up Coolhand on a production AI application, on your own local Claude Code and Copilot skills, or both. For production it checks what your project already has, explains what managed or self-hosted Coolhand requires (API keys, database tables, endpoints, env vars), and hands off to `/feedback-collection`. For local skills it hands off to `/coolhand-capture`. It is a thin alias for the Setup mode in `coolhand-integration`.

`/coolhand-capture` (`/coolhand:coolhand-capture` in the plugin) walks you through capturing your own Claude Code and GitHub Copilot work with the `coolhand` CLI: skill runs as they happen (hooks), past Claude Code sessions on your machine, your local skills, and scheduled skill runs. It does not assume you have a skills repo; it looks, asks, sets up a public API key or a login, and tests that it works.

`/wildcard-tool` is also available as a standalone skill: skill consults on adding a wildcard "complaint box" tool to any agentic project so silent agent failures become visible. See [Your AI Agent Has Notes](https://michael.carroll.io/talks/2026/your-ai-agent-has-notes) for the background on why this matters and how to act on what the tool surfaces.

The plugin is one install — all six skills load together.

It uses the latest SDK documentation from GitHub each time it runs, so recommendations stay current.

## Installation

```sh
skills add Coolhand-Labs/skills -g
```

Don't have `skills`? Install it first: `npm install -g skills`

If you don't have Node, use Claude Code's built-in marketplace:

```
/plugin marketplace add Coolhand-Labs/skills
/plugin install coolhand@coolhand-labs
```

All six skills become available globally across all your projects. The planner (`/feedback-collection`) is the main entry point and dispatches to the implementation skills internally. `/coolhand-setup` is an optional first step, and `/coolhand-capture` and `/wildcard-tool` stand alone.

## Usage

Open Claude Code in your project and run:

```
/feedback-collection
```

To scope the scan to a specific directory:

```
/feedback-collection src/ai/
```

The skill scans your codebase and designs a feedback strategy first, then asks for an API key before setting anything up. For managed Coolhand it needs `COOLHAND_API_KEY` (the skill can open a browser flow to get one via `coolhand login`); for self-hosted it needs `FEEDBACK_API_KEY` and `FEEDBACK_API_KEYS`.

## What makes strong feedback

The planner prioritizes collection strategies in order of signal quality:

| Signal | Field | How it's collected |
|--------|-------|-------------------|
| Best | `revised_output` | User edits the AI output and saves |
| Medium | `explanation` | User writes why the output was good or bad |
| Lowest | `sentiment` (`"like"`/`"dislike"`/`"neutral"`) | Binary accept/reject, buy/skip, thumbs up/down |

It also ensures every feedback item includes a **matching field** so feedback can link to the original LLM call. **At least one of `client_unique_id` or `original_output` is required on every submission as a backstop** — the top two are upgrades when available:

| Match quality | Field | How it's obtained |
|--------------|-------|------------------|
| Best | `llm_request_log_id` | Auto-captured by the Coolhand server SDK |
| Good | `llm_provider_unique_id` | Response body `id` field from LLM API response (e.g., `chatcmpl-…` for OpenAI — **not** the `x-request-id` header) |
| Backstop | `client_unique_id` | Your own session/request/record ID |
| Backstop | `original_output` | The verbatim LLM output for fuzzy matching |

## Supported stacks

- **Ruby** — via [coolhand-ruby](https://github.com/Coolhand-Labs/coolhand-ruby)
- **Python** — via [coolhand-python](https://github.com/Coolhand-Labs/coolhand-python)
- **Node.js / TypeScript** — via [coolhand-node](https://github.com/Coolhand-Labs/coolhand-node)
- **Frontend / Web** — via [coolhand-js](https://github.com/Coolhand-Labs/coolhand-js) widget

## Privacy

The skill enforces safe `creator_unique_id` handling:

- Emails, names, and any directly identifying strings are rejected
- A SHA-256 hash of an internal user ID is always recommended instead
- Consistency across all feedback points in the codebase is checked before implementation

## Transparency

This plugin is open source. You can read exactly what instructions Claude receives:

- Planner: [`skills/feedback-collection/SKILL.md`](skills/feedback-collection/SKILL.md)
- Managed-Coolhand integration and setup walkthrough: [`skills/coolhand-integration/SKILL.md`](skills/coolhand-integration/SKILL.md) (`/coolhand-setup` is an alias: [`skills/coolhand-setup/SKILL.md`](skills/coolhand-setup/SKILL.md))
- Self-hosted: [`skills/self-hosted-feedback/SKILL.md`](skills/self-hosted-feedback/SKILL.md) (with a Coolhand v2 API reference at [`skills/self-hosted-feedback/references/api-spec.md`](skills/self-hosted-feedback/references/api-spec.md))
- Wildcard-tool consultant: [`skills/wildcard-tool/SKILL.md`](skills/wildcard-tool/SKILL.md)
- Local skills and sessions capture walkthrough: [`skills/coolhand-capture/SKILL.md`](skills/coolhand-capture/SKILL.md)

## Issues and feedback

Found a bug or have a suggestion for this skill? [Open an issue](https://github.com/Coolhand-Labs/skills/issues/new).

If the skill surfaces a problem with one of the Coolhand SDKs, it will offer to help you draft a sanitized issue report for the appropriate repository.

## License

Apache-2.0 — see [LICENSE](LICENSE).
