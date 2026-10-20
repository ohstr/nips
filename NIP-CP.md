NIP-CP
======

Cash Pocket
-----------

`draft` `optional`

**Depends on**: [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) (Encrypted Payloads), [NIP-59](https://github.com/nostr-protocol/nips/blob/master/59.md) (Gift Wrap), [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-17](https://github.com/nostr-protocol/nips/blob/master/17.md) (Private Direct Messages), [NIP-CZ](./NIP-CZ.md) (Cash Zaps)

## Abstract

A Cash Pocket stores the NIP-CASH cash tokens a wallet holds. Each token is a kind `9525` Pocket Entry, gift-wrapped to the user's own pubkey, so relays see only a gift wrap addressed to the user, like a DM.

## Motivation

1. **Lost devices**: held cash survives losing the device that received it.
2. **Every device**: the same balance wherever the user signs in.
3. **No tracking**: nobody can tell who holds cash, how many notes, or when they move.

## Terminology

- **Pocket**: all of a user's Pocket Entries.
- **Pocket Entry**: a kind `9525` rumor describing one held token.
- **wrap key**: the one-time secret key that signed the gift wrap carrying an entry.

## Specification

### Event Kinds

| Kind | Name | Signed by | Published |
|------|------|-----------|-----------|
| `9525` | Pocket Entry | nobody (rumor) | only inside a seal and gift wrap |

### Kind 9525 — Pocket Entry

```json
{
  "kind": 9525,
  "pubkey": "<user-pubkey>",
  "created_at": 1760000000,
  "content": "",
  "tags": [
    ["w", "<token-wallet-pubkey>"],
    ["v", "1"],
    ["cash", "satscash1..."],
    ["amount", "21000"],
    ["chain", "bitcoin"],
    ["identity", "pubkey", "<user-pubkey>"],
    ["state", "live", "1760000000"],
    ["mint", "<minter-node-pubkey>"],
    ["expires_at", "1790000000"],
    ["source", "<9522-event-id>"],
    ["wrap_key", "<hex-secret-key-of-this-entry's-wrap>"]
  ]
}
```

**Required tags**:
- `w`: the token's wallet pubkey (TLV `0`), identifying the note.
- `v`: a positive integer, raised by one on every change.
- `cash`: the cash token.
- `amount`: the slice amount in the chain's milli-unit (`msats` for `bitcoin`).
- `chain`: the coin behind the token, e.g. `bitcoin`. Works the same for any energy-backed coin NIP-CASH supports.
- `identity`: who can claim the slice — `["identity", "pubkey", "<pubkey>"]`, `["identity", "connection_key", "<ConnectionKey>", "<ia-pubkey>"]` or `["identity", "cash"]`.
- `state`: `["state", "<state>", "<checked-at>"]`, with `<state>` one of `live`, `spent`, `expired`, `reclaimed`, `unknown`, and `<checked-at>` the unix seconds of the last `cash_status` answer.
- `wrap_key`: the hex secret key of the gift wrap carrying this entry.

**Optional tags**: `mint` (the Minter, NIP-CZ), `expires_at` (from the mint response), `source` (the `9522` it arrived in), `secret` (a `cash`-mode slice's `cash_secret`).

### Wrapping

An entry is sealed (kind `13`, signed by the user) and gift-wrapped (kind `1059`, `p` = the user) per NIP-59. The wallet generates the wrap key first and records it in the entry's `wrap_key`.

Wraps go to the user's NIP-17 inbox relays (kind `10050`), else their NIP-65 write relays. Those relays SHOULD serve a `1059` only to its `p` (NIP-42 AUTH).

## Writing

- **Add** an entry when the wallet receives or mints a token it keeps.
- **Change** an entry by publishing a new wrap with `v` raised by one, then deleting the previous wrap with a NIP-09 kind `5` signed by that wrap's `wrap_key`.
- **Remove** an entry by deleting its wrap. A wallet MAY first keep it for up to 30 days in a final state (`spent`, `expired`, `reclaimed`), so the user can see why a note left.

## Reading

- **Full sync**: `{"kinds": [1059], "#p": ["<user-pubkey>"]}`, then unwrap and keep rumors of kind `9525`. Other wraps (DMs) are ignored.
- **Incremental sync**: the same filter with `since` set to the last sync minus 2 days, since gift wraps carry randomized timestamps. Wraps already seen are skipped.
- **Resolving**: the highest `v` per `w` wins; on a tie, the latest rumor `created_at`.
- **Resync**: drop the local copy and run a full sync.

## Balance

The balance is the sum of `amount` over entries in state `live`. A wallet refreshes states with `cash_status` (NIP-CASH §Cash Status), batched per Hub, and updates entries whose state changed. No answer means `unknown`, never `spent`. Where the Hub quotes it, a wallet SHOULD show `net_redeemable_millis` rather than `amount`.

## Sweeping and Spending

- A wallet SHOULD claim received cash into the user's own Hub promptly; a same-node claim is fee-free. The entry then becomes `spent`. The Pocket is for users without a Hub, and for cash they choose to carry.
- To pay, a wallet SHOULD spend held tokens first (`cash_transfer`, whole or split), then mint from the user's Hub. A split leaves a new token for the remainder, which gets its own entry.
- If two devices spend the same token, the Hub accepts the first; the other device sees the token `spent` on its next refresh.

## Security Considerations

- **Custody**: a held token stays in its Minter's custody. Until claimed, the sender can revoke it (NIP-CZ §Security Considerations).
- **Bearer secrets**: an entry with `secret` is spendable by anyone who reads it. Anyone who can decrypt the user's seals can take it.
- **Stale wraps**: a wallet that fails to delete an old wrap leaves a stale copy behind. The highest `v` still wins.

## Privacy Considerations

Relays see gift wraps addressed to the user at randomized times, the same as DMs, and deletions signed by one-time keys. The number of wraps still grows with activity; a wallet MAY batch changes to hide it. Refreshing states contacts each Hub, not relays.

## Relation to Other NIPs

**[NIP-CASH](./NIP-CASH.md)**: defines the tokens a Pocket holds and the `cash_status` that refreshes them.

**[NIP-CZ](./NIP-CZ.md)**: where most held cash arrives; its `9522` is the entry's `source`.

**[NIP-59](https://github.com/nostr-protocol/nips/blob/master/59.md)** and **[NIP-17](https://github.com/nostr-protocol/nips/blob/master/17.md)**: the wrapping and inbox relays reused here.

**[NIP-60](https://github.com/nostr-protocol/nips/blob/master/60.md)** (Cashu Wallet): the same role for Cashu.
