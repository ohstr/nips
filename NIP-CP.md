NIP-CP
======

Cash Pocket
-----------

`draft` `optional`

**Depends on**: [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) (Encrypted Payloads), [NIP-59](https://github.com/nostr-protocol/nips/blob/master/59.md) (Gift Wrap), [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-17](https://github.com/nostr-protocol/nips/blob/master/17.md) (Private Direct Messages), [NIP-CZ](./NIP-CZ.md) (Cash Zaps)

## Abstract

A Cash Pocket stores the cash tokens a wallet holds. Each token is a kind `9525` entry, gift-wrapped to the user's own pubkey: relays see only what looks like a DM, with no author, kind or amount.

## Motivation

1. **Lost devices**: held cash survives.
2. **Every device**: one balance everywhere.
3. **No tracking, no extra key**: nobody sees who holds cash or how much, and the user's own key both claims and stores it.

## Terminology

- **Pocket**: a user's Pocket Entries.
- **Pocket Entry**: one held token, as a kind `9525` rumor.
- **wrap key**: the one-time key of an entry's gift wrap.

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
- `w`: token wallet pubkey (TLV `0`).
- `v`: version, +1 on every change.
- `cash`: the token.
- `amount`: milli-units (`msats` for `bitcoin`).
- `chain`: e.g. `bitcoin`; any energy-backed coin works the same.
- `identity`: who can claim it — `pubkey <pubkey>`, `connection_key <ConnectionKey> <ia-pubkey>` or `cash`.
- `state`: `live`, `pending`, `spent`, `expired`, `reclaimed` or `unknown`, plus the last check time.
- `wrap_key`: secret key of this entry's gift wrap, in hex.
- `expires_at`: when the bill returns to its minting Hub, in unix seconds; required for a bill that expires, taken from the Hub's reply, `cash_status` or the `9522`.

A `pending` entry leaves out `cash` and `identity` and carries `request_id` instead (§Writing).

**Optional tags**: `mint` (Minter), `source` (its `9522`), `secret` (`cash_secret` of a `cash`-mode token), `request_id` (of the call that creates the token).

**Published**: only inside a seal and gift wrap. A `9525` MUST NOT be signed or published on its own. Relays SHOULD reject a signed `9525`.

### Wrapping

An entry is sealed (kind `13`, signed by the user) and gift-wrapped (kind `1059`, `p` = the user) per NIP-59. The wallet generates the wrap key first and records it in the entry's `wrap_key`.

Wraps go to the user's NIP-17 inbox relays (kind `10050`), else their NIP-65 write relays. Those relays SHOULD serve a `1059` only to its `p` (NIP-42 AUTH).

## Writing

- **Add** an entry when the wallet receives or mints a token it keeps.
- **Write ahead**: before any call that creates a bill (`mint_cash`, `cash_transfer`, `cash_consolidate`), the wallet saves a `pending` entry holding the call's `request_id` (NIP-CASH) and any cash secret it generated, with `w` set to the `request_id`. On the reply it adds an entry for each new bill and removes the pending one. If the reply is lost, the wallet recovers it with `cash_lookup` and MUST NOT resend the call.
- **Change** an entry by publishing a new wrap with `v` raised by one, then deleting the previous wrap with a NIP-09 kind `5` signed by that wrap's `wrap_key`.
- **Remove** an entry by deleting its wrap. A wallet MAY first keep it for up to 30 days in a final state (`spent`, `expired`, `reclaimed`), so the user can see why a note left.
- **Expiration**: a wallet MAY set a NIP-40 `expiration` on the gift wrap: `expires_at` plus a grace period (e.g. 30 days), rounded up to the week plus a random number of days. It still deletes the wrap itself, since relays may ignore NIP-40.

## Reading

- **Full sync**: `{"kinds": [1059], "#p": ["<user-pubkey>"]}`, then unwrap and keep rumors of kind `9525` whose seal `pubkey` and rumor `pubkey` are both the user. Other wraps (DMs) are ignored. A wallet MUST discard any other `9525`: anyone can gift-wrap to the user, but only the user can sign the seal.
- **Incremental sync**: the same filter with `since` set to the last sync minus 2 days, since gift wraps carry randomized timestamps. Wraps already seen are skipped.
- **Resolving**: per `w`, a final state (`spent`, `expired`, `reclaimed`) wins over `live`, `unknown` or `pending`, whatever its `v`. Otherwise the highest `v` wins; on a tie, the latest rumor `created_at`. On any conflict the wallet re-checks the bill with `cash_status`.
- **Resync**: drop the local copy and run a full sync.

## Balance

The balance, per `chain`, is the sum of `amount` over entries in state `live`. A wallet refreshes states with `cash_status` (NIP-CASH §Cash Status), batched per Hub, and updates entries whose state changed:

| Result | State |
|--------|-------|
| the wallet's own spend succeeded | `spent` |
| a tombstone after `expires_at` | `expired` |
| `NOT_FOUND` on a live bill | `reclaimed` |
| any other tombstone | `spent` |
| no answer | `unknown`, never `spent` |

A device MUST write a final state only from a definitive result (its own spend, a tombstone, `NOT_FOUND`), never from a timeout or a cached view. Where the Hub quotes it, a wallet SHOULD show `net_redeemable_millis` rather than `amount`.

## Sweeping and Spending

- A wallet SHOULD claim received cash into the user's own Hub promptly; a same-node claim is fee-free. The entry then becomes `spent`. The Pocket holds cash until claimed; a held bill returns to its minting Hub at `expires_at`.
- A wallet MUST show `expires_at`, and SHOULD warn and claim or redeem before it.
- To pay, a wallet SHOULD spend held tokens first (`cash_transfer`, whole or split), then mint from the user's Hub. A split leaves a new token for the remainder, which gets its own entry.
- If two devices spend the same token, the Hub accepts the first; the other device sees the token `spent` on its next refresh.

## Security Considerations

- **Custody**: a held token stays in its Minter's custody. Until claimed, the sender can revoke it (NIP-CZ §Security Considerations).
- **Bearer secrets**: an entry with `secret` is spendable by anyone who reads it. Anyone who can decrypt the user's seals can take it.
- **Stale wraps**: a wallet that fails to delete an old wrap leaves a stale copy behind. Resolving still picks the current one.

## Privacy Considerations

Relays see gift wraps addressed to the user at randomized times, the same as DMs, and deletions signed by one-time keys. The number of wraps still grows with activity; a wallet MAY batch changes to hide it. Refreshing states contacts each Hub, not relays.
