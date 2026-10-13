# ohstr NIPs

Nostr Improvement Proposals (NIPs) maintained by ohstr.

| NIP | Title | Kinds | Status |
|-----|-------|-------|--------|
| [NIP-AZ](./NIP-AZ.md) | AltZap: Cross-Chain Lightning Zaps | `5520`, `5521`, `5523` | `draft` |
| [NIP-IC](./NIP-IC.md) | Identity Connection: Legacy Account Linking | `35521`, `35522` | `draft` |

**NIP-AZ** extends [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) so a zap can settle on any energy-backed Lightning-routable chain (not just Bitcoin) and target a recipient who doesn't have a Nostr keypair yet.

**NIP-IC** defines how a user cryptographically links their Nostr pubkey to an account on a non-Nostr platform (Discord, X, email, ...), witnessed by a permissionless network of Identity Authorities — the mechanism NIP-AZ relies on to resolve a `ConnectionKey` back to a verified identity.
