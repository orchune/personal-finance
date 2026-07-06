# Orchune Personal Finance Skill

An [Agent Skill](https://agentskills.io) that lets AI agents (Claude Code, Codex, etc.) record and query your personal finances through the [Orchune](https://www.orchune.com) MCP server — log expenses and income, import bank/credit-card statements, track budgets and savings goals, manage accounts and categories, and record stock/option/fund trades.

## Install

Claude Code / Codex (via [skills.sh](https://www.skills.sh)):

```bash
npx skills add orchune/personal-finance
```

Hermes Agent:

```bash
hermes skills install orchune/personal-finance/skills/personal-finance
```

OpenClaw (via [ClawHub](https://docs.openclaw.ai/clawhub)):

```bash
clawhub skill install personal-finance
```

## What it does

- **Bookkeeping**: log expenses, income, and transfers in natural language
- **Statement import**: import bank, credit-card, and payment platform bills
- **Insights**: check spending, budgets, and savings goals
- **Investments**: record stock, option, and fund trades
- **Auth built in**: sign up or log in with an emailed 6-digit code — no manual token setup required

## Requirements

An MCP client with Streamable HTTP support and network access to `https://www.orchune.com/mcp`.

## License

Proprietary — see [SKILL.md](skills/personal-finance/SKILL.md) for details.
