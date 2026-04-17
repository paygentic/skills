# Paygentic AI Skills

Claude Code plugin that helps developers integrate [Paygentic](https://paygentic.io) — a consumption-based billing platform for SaaS, AI, and API companies. It handles everything between "customer used your product" and "customer gets charged correctly": usage metering, entitlements, subscriptions, invoicing, and payments.

## What This Plugin Does

Instead of reading docs and writing boilerplate, describe what you need in natural language and the skills generate correct, production-ready integration code.

| Skill | What it does |
|-------|-------------|
| `explore` | Explains Paygentic concepts — billing models, metering, entitlements, credit-based billing — so you can evaluate the platform and plan your integration without writing code |
| `integrate` | Generates working SDK code (TypeScript or Python) for usage metering, customer management, subscriptions, entitlements, invoices, and payments. Auto-detects your language from the project |

### Examples

```
"How does credit-based billing work in Paygentic?"     → explore skill
"Send usage events from my Express API"                → integrate skill (TypeScript)
"Check if a customer has API calls remaining in Python" → integrate skill (Python)
"Set up full billing for my SaaS"                      → integrate skill (full setup)
```

## Installation

```bash
/plugin marketplace add paygentic/skills
/plugin install paygentic@paygentic-skills
```

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- Paygentic account with API key — sign up at [platform.paygentic.io](https://platform.paygentic.io)
- One of the Paygentic SDKs:

  | Language | Package | Install |
  |----------|---------|---------|
  | TypeScript | `@paygentic/sdk` | `npm install @paygentic/sdk` |
  | Python | `paygentic-sdk` | `pip install paygentic-sdk` |

## Eval Results

Benchmarked across 5 evaluations (20 assertions) comparing Claude with and without the plugin:

| | With skill | Without skill | Delta |
|---|---|---|---|
| **Accuracy** | 90% (18/20) | 75% (15/20) | +15% |
| **Tokens** | ~22K avg | ~63K avg | -65% |
| **Speed** | 46s avg | 120s avg | 2.6x faster |

Biggest impact: usage metering (eval 3) — without the skill, Claude defaults to deprecated v0 endpoints (`usageEvents.create()`) that error on v1 billing plans. The skill prevents this entirely.

## Updating

```bash
/plugin marketplace update paygentic-skills
/plugin update paygentic@paygentic-skills
```

## Resources

- [Documentation](https://docs.paygentic.io)
- [API reference](https://docs.paygentic.io/api-reference)
- [Quickstart guide](https://docs.paygentic.io/getting-started/quickstart)
