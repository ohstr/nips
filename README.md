# NIP-AZ (AltZap) Agent Skills ⚡️

AI Agent skills for building, debugging, and integrating **AltZap** (NIP-AZ) — the [Zapf](https://docs.zapf.app) protocol extension for cross-chain Lightning zaps.

This repository contains machine-readable markdown instructions (`SKILL.md`) designed to be consumed by agentic coding tools like Claude Code, Cursor, and Copilot Workspace.

## Documentation

Full protocol coverage lives in the [Zapf docs](https://docs.zapf.app):

- [Introduction](https://docs.zapf.app/intro) — architecture overview and core concepts
- [AltZap Request — Kind 5520](https://docs.zapf.app/protocol/zap-request-5520)
- [AltZap Receipt — Kind 5521](https://docs.zapf.app/protocol/zap-receipt-5521)
- [On-Behalf AltZap Request — Kind 5523](https://docs.zapf.app/protocol/on-behalf-zap-request-5523)
- [AltZap Settlement Provider (ZSP)](https://docs.zapf.app/concepts/zap-settlement-provider)

The reference Go implementation lives in [`github.com/ohstr/nmilat`](https://github.com/ohstr/nmilat)'s `nipAZ` package.

## Installation

You can install these skills directly into your project using the [Agent Skills CLI](https://skills.sh/).

To install the core NIP-AZ skill:

```bash
npx skills add ohstr/nip-az
```

This will download the skill into your local `.agents/skills/nip-az` directory, giving your AI assistant instant expertise on:
- Kind 5520 (AltZap Requests) and 5521 (AltZap Receipts)
- Fallback Lightning Addresses and LNURL Webhooks
- Identity Resolution workflows

## Usage

Once installed, simply ask your AI agent to implement a feature related to AltZap, and it will automatically reference the installed skill for the canonical rules, code examples, and best practices.

*Example prompt:*
> "Create a new function to generate a Kind 5520 AltZap Request event according to the nip-az skill."
