# nips

Nostr Improvement Proposals (NIPs) published by ohstr. Each NIP is self-contained: it specifies a protocol on its own terms and can be implemented without anything else in this repository.

## NIPs in this repository

| NIP | Title | Kinds | Status |
|-----|-------|-------|--------|
| [NIP-AZ](./NIP-AZ.md) | AltZap: Cross-Chain Lightning Zaps | `9520`, `9521`, `9523` | `draft` |
| [NIP-CASH](./NIP-CASH.md) | Cash Hub | `11190`, `23190`–`23193` | `draft` |
| [NIP-CP](./NIP-CP.md) | Cash Pocket | `9525` | `draft` |
| [NIP-CW](./NIP-CW.md) | Circle Wallet | `23199` | `draft` |
| [NIP-CZ](./NIP-CZ.md) | Cash Zaps | `9522`, `10522` | `draft` |
| [NIP-IC](./NIP-IC.md) | Identity Connection: Web Identity Linking | `35521`, `35522` | `draft` |
| [NIP-LS](./NIP-LS.md) | Local Signer: NIP-46 over a Unix Socket | — | `draft` |

**NIP-AZ**: zaps that settle on any energy-backed Lightning chain.

**NIP-CASH**: ecash for energy-backed Lightning coins, as cash tokens a Hub mints and holders hand over, split and redeem.

**NIP-CP**: a private, multi-device store for the cash tokens a wallet holds.

**NIP-CW**: shared wallets a Hub issues to the members of a circle.

**NIP-CZ**: zaps paid with a cash token instead of a Lightning invoice.

**NIP-IC**: links a Nostr pubkey to a Web Identity, attested by Identity Authorities.

**NIP-LS**: NIP-46 signing over a local unix socket, with no relay.
