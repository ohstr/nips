# nips

Nostr Improvement Proposals (NIPs) published by ohstr. Each NIP is self-contained: it specifies a protocol on its own terms and can be implemented without anything else in this repository.

## NIPs in this repository

| NIP | Title | Kinds | Status |
|-----|-------|-------|--------|
| [NIP-AZ](./NIP-AZ.md) | AltZap: Cross-Chain Lightning Zaps | `5520`, `5521`, `5523` | `draft` |
| [NIP-IC](./NIP-IC.md) | Identity Connection: Legacy Account Linking | `35521`, `35522` | `draft` |
| [NIP-LS](./NIP-LS.md) | Local Signer: NIP-46 over a Unix Socket | — | `draft` |

**NIP-AZ** extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) so a zap can settle on any energy-backed Lightning-routable chain (not just Bitcoin) and target a recipient who doesn't have a Nostr keypair yet.

**NIP-IC** defines how a user cryptographically links their Nostr pubkey to an account on a non-Nostr platform (Discord, X, email, ...), witnessed by a permissionless network of Identity Authorities — the mechanism NIP-AZ relies on to resolve a `ConnectionKey` back to a verified identity.

**NIP-LS** lets a process on the same host sign through a key it can't read: a local signer answers the [NIP-46](https://github.com/nostr-protocol/nips/blob/master/46.md) method set as newline-delimited JSON over a unix socket (`bunker+unix:///path.sock`), with no relay and no encryption, and its own policy decides what gets signed.

## Implementations

Go: [`github.com/ohstr/nmilat`](https://github.com/ohstr/nmilat), packages `nipAZ`, `nipIC` and `nipLS`.
