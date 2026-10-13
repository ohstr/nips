# nips

Nostr Improvement Proposals (NIPs) published by ohstr. Each NIP is self-contained: it specifies a protocol on its own terms and can be implemented without anything else in this repository.

## NIPs in this repository

| NIP | Title | Kinds | Status |
|-----|-------|-------|--------|
| [NIP-AZ](./NIP-AZ.md) | AltZap: Cross-Chain Lightning Zaps | `9520`, `9521`, `9523` | `draft` |
| [NIP-CASH](./NIP-CASH.md) | Cash Hub | `11190`, `23190`–`23193` | `draft` |
| [NIP-CW](./NIP-CW.md) | Circle Wallet | `23199` | `draft` |
| [NIP-IC](./NIP-IC.md) | Identity Connection: Web Identity Linking | `35521`, `35522` | `draft` |
| [NIP-LS](./NIP-LS.md) | Local Signer: NIP-46 over a Unix Socket | — | `draft` |

**NIP-AZ** extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) so a zap can settle on any energy-backed Lightning-routable chain (not just Bitcoin) and target a recipient who doesn't have a Nostr keypair yet.

**NIP-CASH** is a Chaumian ecash system for energy-backed Lightning coins: a Hub mints cash tokens (`lokicash1...`, `satscash1...`) as specially-scoped [NIP-47](https://github.com/nostr-protocol/nips/blob/master/47.md) connections that can be handed over, split, consolidated and redeemed, with a private transport for moving bills.

**NIP-CW** defines Circle Wallets: shared, self-service NIP-47 wallets a Hub issues to members of a circle, who prove membership with a signed identity event.

**NIP-IC** defines how a user cryptographically links their Nostr pubkey to an account on a non-Nostr platform (Discord, X, email, ...), witnessed by a permissionless network of Identity Authorities — the mechanism NIP-AZ relies on to resolve a `ConnectionKey` back to a verified identity.

**NIP-LS** lets a process on the same host sign through a key it can't read: a local signer answers the [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) method set as newline-delimited JSON over a unix socket (`bunker+unix:///path.sock`), with no relay and no encryption, and its own policy decides what gets signed.
