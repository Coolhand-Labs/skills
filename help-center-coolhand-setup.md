---
title: "Set up Coolhand with the /coolhand-setup skill"
answer: "Run /coolhand-setup in Claude Code before you install anything. It checks what your project already has, asks whether you want managed or self-hosted Coolhand, lists the API keys, database tables, endpoints and environment variables that path needs, and then hands you to /feedback-collection to do the actual integration."
category: "Coding agents"
tags: ["setup","skills","claude-code","api-keys","self-hosted"]
---

# Set up Coolhand with the /coolhand-setup skill

Run `/coolhand-setup` in Claude Code before you install anything. It checks what your project already has, asks whether you want managed or self-hosted Coolhand, lists the API keys, database tables, endpoints and environment variables that path needs, and then hands you to `/feedback-collection` to do the actual integration.

---

## Install the skills

The skills ship as one plugin from the Coolhand Labs skills repository.

```sh
skills add Coolhand-Labs/skills -g
```

If you do not have Node, use Claude Code's built-in marketplace instead:

```
/plugin marketplace add Coolhand-Labs/skills
/plugin install coolhand@coolhand-labs
```

When installed as a plugin, the command is `/coolhand:coolhand-setup`. Installed with the `skills` CLI, it is `/coolhand-setup`.

Repository: https://github.com/Coolhand-Labs/skills

---

## What /coolhand-setup does

It is a readiness check. It does not install SDKs, write database migrations or create endpoints. The only change it can make to your project is adding an API key to an env file, and it asks you first.

1. **Inspects your project.** It looks for Coolhand environment variable names in your `.env*` files (it reports names and file paths, never values), checks whether you are logged in with `coolhand status`, looks for an existing Coolhand integration, detects your database and ORM, and notes which LLM provider SDKs you use.
2. **Asks which path you want.** Managed sends your logs and feedback to coolhandlabs.com. Self-hosted sends them to endpoints on your own backend.
3. **Shows a checklist.** Each item is marked found, missing or not needed, with what creates it.
4. **Gets missing keys, with your approval.**
5. **Hands off** to `/feedback-collection`, which scans your code for LLM calls, proposes a feedback strategy for your approval, and then sets everything up.

You can skip the question by passing the path: `/coolhand-setup managed` or `/coolhand-setup self-hosted`.

---

## What the managed path needs

| Item | Notes |
|---|---|
| A Coolhand account and `COOLHAND_API_KEY` | The skill gets this for you with `coolhand login --write-env <file>`, which opens a browser to authorize. You choose which env file it writes to. |
| `COOLHAND_PRIVATE_KEY` | Only if you want to read data back or use optimization commands. Server-side and CLI use only. Never put it in frontend code or commit it. |
| The Coolhand server SDK | Installed later by the integration step, not by setup. |
| The coolhand-js widget | Only if you approve adding a feedback widget to your UI. |
| Anything in your database | Not needed. |

---

## What the self-hosted path needs

Self-hosted means you own the data and the analysis. You need:

| Item | Notes |
|---|---|
| A database with migration tooling | Required. Without one, setup recommends managed. |
| Two tables: `llm_request_logs` and `llm_request_log_feedbacks` | Generated as a migration in your ORM's syntax by the self-hosted integration step. |
| Three endpoints: `POST /api/v2/llm_request_logs`, `POST /api/v2/llm_request_log_feedbacks` and `PATCH /api/v2/llm_request_log_feedbacks/{id}` | Mirror the Coolhand v2 API. |
| Auth middleware | Accepts the keys listed in `FEEDBACK_API_KEYS`. |
| `FEEDBACK_API_KEY` and `FEEDBACK_API_KEYS` | One opaque secret, set in both places: the SDK sends it, and the server must list it as accepted. `FEEDBACK_API_KEYS` is comma-separated so you can add more clients later. |
| `FEEDBACK_BASE_URL` | The https URL of your host, reachable from wherever the SDK runs. You provide this. |
| An SDK release with `base_url` support | Setup tells you which SDKs have shipped it and which are still pending. |

The exact table columns and endpoint shapes are in the self-hosted skill's API reference in the skills repository, so setup does not repeat them.

Moving from managed to self-hosted later means starting fresh: the v2 API does not yet have export endpoints, so historical data does not carry over.

---

## Troubleshooting

**`coolhand status` says I am not logged in.** That is expected on a new machine. Setup treats it as "no key yet" and offers the browser login.

**Setup says there is nothing for Coolhand to collect.** It found no LLM provider SDK imports in the folder it checked. Run it from your project root, or continue anyway if your LLM calls are made over plain HTTP.

**I picked self-hosted and it recommended managed.** Self-hosted needs somewhere to store data. Add a database, or start on managed and re-run `/coolhand-setup self-hosted` later.

**My key is already in `.env` but setup lists it as missing.** It only reads the standard env file names (`.env`, `.env.local`, `.env.development`, `.env.production` and other `.env*` files) and the exact variable names above. Check the spelling, or tell it which file holds the key.

---

## Related

- Help Center index: https://coolhandlabs.com/help.md
- Skills repository: https://github.com/Coolhand-Labs/skills
