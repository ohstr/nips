# zapf-nips

Nostr Improvement Proposals (NIPs) originating from the [Zapf](https://docs.zapf.app) project, plus an installable [Agent Skills](https://skills.sh/) package (`SKILL.md` + `references/`) so AI coding assistants can build against them directly.

## NIPs in this repository

| NIP | Title | Kinds | Status |
|-----|-------|-------|--------|
| [NIP-AZ](./NIP-AZ.md) | AltZap: Cross-Chain Lightning Zaps | `5520`, `5521`, `5523` | `draft` |
| [NIP-IC](./NIP-IC.md) | Identity Connection: Legacy Account Linking | `35521`, `35522` | `draft` |

**NIP-AZ** extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) so a zap can settle on any energy-backed Lightning-routable chain (not just Bitcoin) and target a recipient who doesn't have a Nostr keypair yet.

**NIP-IC** defines how a user cryptographically links their Nostr pubkey to an account on a non-Nostr platform (Discord, X, email, ...), witnessed by a permissionless network of Identity Authorities — the mechanism NIP-AZ relies on to resolve a `ConnectionKey` back to a verified identity.

Reference implementation: [`github.com/ohstr/nmilat`](https://github.com/ohstr/nmilat), package `nipAZ` (NIP-IC has no standalone library yet — see its own doc for details).

## Installation (Agent Skill)

```bash
npx skills add ohstr/zapf-nips
```

This downloads `SKILL.md` and `references/` into your local `.agents/skills/zapf-nips` directory, giving your AI assistant instant, sourced expertise on both NIPs — event kinds, tag formats, validation rules, and identity resolution.

*Example prompt:*
> "Create a new function to generate a Kind 5520 AltZap Request event according to the zapf-nips skill."

## Documentation

Deployed protocol docs and guides: [docs.zapf.app](https://docs.zapf.app)
