# Dailybot Plugin for Claude Code

Connect [Claude Code](https://code.claude.com) to your team via [Dailybot](https://www.dailybot.com). Report progress, check messages, send emails, and announce agent status — all integrated into your coding workflow.

## What it does

This plugin gives Claude Code five capabilities for team collaboration through Dailybot:

**Progress Reporting** — When you complete meaningful work (shipping a feature, fixing a bug, finishing a task), the plugin sends a standup-style progress report to Dailybot. Reports are written in a human-first style: they describe what was accomplished and why it matters, not which files changed.

**Team Messages** — Check for pending messages and instructions from your team. Messages represent tasks, priorities, context, or feedback that should influence your current work.

**Email** — Send emails to anyone through Dailybot. Useful for notifications, summaries, follow-ups, or weekly reports. Replies are delivered back as messages to your agent inbox.

**Health Status** — Announce your agent's status (online, working, offline, degraded) so the team knows what's happening. Health checks also deliver any pending messages from the team.

**Guided Setup** — First-time setup walks you through CLI installation, authentication, and agent profile configuration step by step.

## Install

From inside Claude Code, register the DailyBot marketplace once and install the plugin:

```
/plugin marketplace add DailyBotHQ/dailybot-claude-plugin
/plugin install dailybot@dailybot
```

The first command tells Claude Code to trust this repository as a plugin source. The second installs the `dailybot` plugin from that marketplace. From then on you can update with `/plugin update dailybot@dailybot`.

Or test locally during development without registering a marketplace:

```bash
claude --plugin-dir ./path/to/dailybot-claude-plugin
```

## Setup (one time)

### 1. Dailybot CLI

The plugin requires the Dailybot CLI. Run `/dailybot:setup` for guided setup, or install it yourself:

```bash
# Option A: pip
pip install dailybot-cli

# Option B: install script
curl -sSL https://cli.dailybot.com/install.sh | bash
```

Requires Python 3.9+.

### 2. Authentication

Claude will guide you through login on first use. Or do it yourself:

```bash
dailybot login
```

This sends a verification code to your email. Any Dailybot team member can log in — you don't need to be an admin.

**Don't have a Dailybot account yet?** No problem — Claude can create a new Dailybot organization for you right from the terminal. Just tell Claude you don't have an account and it will handle the setup. You'll get a link to share with your team so they can connect Dailybot to Slack, Teams, Discord, or Google Chat.

### 3. Agent name (optional)

When you enable the plugin, Claude Code asks for an agent name. This is how your reports appear in Dailybot. Default is `claude-code`. You can change it to anything descriptive like `my-backend-agent` or `deploy-bot`.

## Usage

### Slash commands

| Command | What it does |
|---------|-------------|
| `/dailybot:report` | Send a progress report to your team |
| `/dailybot:messages` | Check for pending messages from your team |
| `/dailybot:email` | Send an email through Dailybot |
| `/dailybot:health` | Announce agent status and pick up messages |
| `/dailybot:setup` | Guided first-time setup (CLI install, auth, agent profile) |

### Natural language

You can also use natural language:

- "Send an update to Dailybot about what we did"
- "Report this to my team"
- "Do I have any messages?"
- "What should I work on next?"
- "Email this summary to alice@company.com"
- "Go online" / "Check in with the team"

### Automatic behavior

- **Progress reporting**: Claude detects when you've completed significant work and sends a report automatically. Trivial changes (typo fixes, lockfile updates, formatting) are skipped.
- **Stop gate**: When a turn completes with significant unreported work, Claude is reminded to offer a Dailybot report. The reminder is rate-limited (at most once per 60 seconds) and suppressed once a report is sent.
- **Message check**: At the start of each session, pending messages from your team are fetched automatically.

## Hook architecture

The plugin uses a layered hook design to keep things invisible during normal use and only surface a nudge when there's real work to report.

| Event | Script | Role |
|-------|--------|------|
| `SessionStart` | (inline) | Fetch pending team messages |
| `PostToolUse` (Bash) | `hooks/mark-reported.sh` | Detect successful report commands (`dailybot agent update` or HTTP API) and silently create a `.reported` flag so the Stop gate does not re-nudge. Always silent. |
| `PostToolUse` (Edit/Write/MultiEdit/Bash) | `hooks/accumulator.sh` | Silently log meaningful tool uses to a per-session file. Filters lockfiles, `node_modules/`, `dist/`, `.git/`, `build/`, etc. Always silent. |
| `Stop` | `hooks/gate.sh` | Run cheap deterministic gates (loop guard, already-reported flag, 60s rate-limit, ≥2 edits, ≥60s session age, auth check). Emits a validated `decision`/`reason` JSON payload for the model — never a prompt body, never reasoning. |
| `SessionEnd` | `hooks/cleanup.sh` | Remove the session's `.log`, `.reported`, `.last-fired` files |

### Environment variables

| Var | Default | Purpose |
|-----|---------|---------|
| `CLAUDE_PLUGIN_DATA` | `$HOME/.dailybot-claude/sessions` | Storage root for per-session log/flag files |
| `DAILYBOT_DETERMINISTIC_ONLY` | `0` | By default, after the deterministic gates pass, an additional Claude Haiku qualitative check (`gate-llm.sh`) refines the decision. Set to `1` to skip it. If the LLM call fails (missing `ANTHROPIC_API_KEY`, network, malformed response), the gate falls back to the deterministic verdict — the plugin keeps working offline. |
| `DAILYBOT_DEBUG` | `0` | Set to `1` to preserve `.log` files at SessionEnd for inspection |

Session log files older than 7 days are cleaned up automatically on the next `PostToolUse` fire.

## Report examples

**Simple bug fix:**
> "Fixed a bug where users without a timezone set would see errors on their profile page."

**Feature with structured data:**
> "Built the notification preferences system — users can now configure which alerts they receive and through which channels."
> + completed: ["Preferences model", "REST API", "Email integration", "Test suite (32 cases)"]

**Milestone:**
> "Shipped the new billing dashboard — managers can now view usage, invoices, and plan details in one place."

## Co-authoring

When you're logged in via CLI, Dailybot automatically credits you as a co-author on every report. The agent's work shows up in your daily standup — because you directed it.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| "Dailybot CLI not found" | Install with `pip install dailybot-cli` or `curl -sSL https://cli.dailybot.com/install.sh \| bash` |
| "Not authenticated" | Run `dailybot login` and follow the email verification flow |
| No Dailybot account | Tell Claude you don't have an account — it can create a new organization for you on the spot |
| Reports not appearing | Check `dailybot status` to verify authentication. Ensure you're a member of a Dailybot organization. |
| Wrong agent name | Run `/plugin` in Claude Code, find the Dailybot plugin, and update the agent name setting |
| Messages not loading | Check `dailybot agent message list --name claude-code --pending` manually to verify |
| Email rate limited | Dailybot limits agent emails per hour (default: 50). Wait for the hourly window to reset. |

## Links

- [Dailybot](https://www.dailybot.com)
- [Dailybot CLI on PyPI](https://pypi.org/project/dailybot-cli/)
- [Dailybot Agent API docs](https://api.dailybot.com/skill.md)
- [Plugin source code](https://github.com/DailyBotHQ/dailybot-claude-plugin)
