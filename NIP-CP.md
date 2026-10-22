NIP-CP
======

Cash Pocket
-----------

`draft` `optional`

**Depends on**: [NIP-44](https://github.com/nostr-protocol/nips/blob/master/44.md) (Encrypted Payloads), [NIP-59](https://github.com/nostr-protocol/nips/blob/master/59.md) (Gift Wrap), [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-17](https://github.com/nostr-protocol/nips/blob/master/17.md) (Private Direct Messages), [NIP-40](https://github.com/nostr-protocol/nips/blob/master/40.md) (Expiration Timestamp), [NIP-CZ](./NIP-CZ.md) (Cash Zaps)

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
- `w`: token wallet pubkey (TLV `0`), or the `request_id` for a `pending` entry and the `done` entry that closes it.
- `v`: version, +1 on every change.
- `cash`: the token.
- `amount`: milli-units (`msats` for `bitcoin`).
- `chain`: e.g. `bitcoin`; any energy-backed coin works the same.
- `identity`: who can claim it — `pubkey <pubkey>`, `connection_key <ConnectionKey> <ia-pubkey>` or `cash`.
- `state`: `live`, `pending`, `spent`, `expired`, `reclaimed`, `done` or `unknown`, plus the last check time in unix seconds.
- `wrap_key`: secret key of this entry's gift wrap, in hex.
- `expires_at`: when the bill returns to its minting Hub, in unix seconds; required for a bill that expires, taken from the Hub's reply, `cash_status`, or a `9522` whose zap signature verifies (NIP-CZ rule 7), never from any other `9522`, whose `expires_at` the sender chose. A bill that never expires has no `expires_at` tag; a `0` in a reply means never and is not stored.

A `pending` entry carries `w`, `v`, `state`, `wrap_key`, `request_id`, `deadline`, `target` and `device` (§Writing), and the call's `amount` and `chain`. It leaves out `cash`, `identity` and `expires_at`. Its `w` equals its `request_id`.
- `request_id`: the call's NIP-CASH `request_id`.
- `deadline`: the call's deadline as sent (the envelope's `not_after`, or `mint_cash`'s NIP-47 `expiration`).
- `target`: the `w` of the bill the call named (the first source for `cash_consolidate`), or for `mint_cash` the Cash Hub connection's wallet pubkey (its TLV `0`).
- `device`: a random id for the device that wrote the entry, generated on that device and not copied by backup or restore, in lowercase hex.

**Optional tags**: `mint` (Minter), `source` (its `9522`), `secret` (`cash_secret` of a `cash`-mode token).

**Published**: only inside a seal and gift wrap. A `9525` MUST NOT be signed or published outside a seal. Relays SHOULD reject a signed `9525`.

### Wrapping

An entry is sealed (kind `13`, signed by the user) and gift-wrapped (kind `1059`, `p` = the user) per NIP-59. The wallet generates the wrap key first and records it in the entry's `wrap_key`.

Wraps go to the user's NIP-17 inbox relays (kind `10050`), else their NIP-65 write relays. Those relays SHOULD serve a `1059` only to its `p` (NIP-42 AUTH). The user MAY add their own relay (one they run or control) to their `10050` list.

A wallet SHOULD write each wrap to at least 2 relays, read it back before treating it as stored, and on each sync republish entries a relay no longer returns.

## Writing

- **Add** an entry when the wallet receives or mints a token it keeps.
- **Write ahead**: before any call that creates a bill (`mint_cash`, `cash_transfer`, `cash_consolidate`), the wallet stores a `pending` entry holding the call's `request_id`, `deadline`, `target`, its own `device` and any cash secret it generated, and reads it back before sending the call, which carries that `request_id`. On the reply the wallet adds an entry for each new bill and marks the pending entry `done`. If the call is refused with `REQUEST_REUSED`, the wallet keeps the pending entry open and looks it up as below; it never closes it as failed. If the reply is lost and the Hub lists `cash_lookup` in `private_methods`, the wallet sends `cash_lookup` with `request_id` and `deadline` to that target, and MUST NOT resend the call. The Hub answers on its own clock:
  - `done`: add an entry for each new bill (a tombstone in place of a token sets that entry's state per §Balance) and mark the pending entry `done`.
  - `pending`: ask again at `retry_after` (the call's `deadline` plus 1).
  - `unknown`: only the device whose `device` matches the entry MAY place the call again. The re-placed call always uses `request_id` = lowercase hex of sha256("retry:" ‖ the replaced entry's `request_id`), never a fresh id, and is written ahead as a new pending entry; then that device marks the old pending entry `done`. The Hub refuses a derived id only once it has committed it, so the entry check and the local record below are what stop a call being placed twice. Just before placing a call again, the owner device syncs and confirms the old entry is still open. It MUST then check for an entry whose `w` is the derived id, counting every entry with that `w`, whatever its `state`, including one this wallet does not recognise; if one exists, it MUST NOT place the call again and only marks the old entry `done`. The device also keeps a local record of every derived id it places, recorded before sending the call and kept across Resync until that id's `deadline` plus the largest `max_request_age` the Hub has announced since the original call, and never places the same id twice. The `device` id and this record are kept and lost together. Any other device SHOULD show the call to the user and leaves the entry open. An omitted answer is not a lookup `unknown`.
  - `expired`: the outcome can no longer be known (this lookup answer is not the bill state `expired`). The wallet MUST NOT place the call again automatically and SHOULD show the call to the user. The device whose `device` matches the entry marks it `done`; any other device leaves it open.
  - A pending entry shown to the user MAY be marked `done` on the user's decision; the wallet MUST NOT place the call again when doing so.
- **Without `cash_lookup`**: the Hub still refuses a repeated `request_id`, so the pending entry is written the same way, but a lost reply needs the Hub operator.
- **Old Hubs**: on a Hub that does not announce `max_request_age`, the wallet treats `request_id` as unhonoured and MUST NOT resend a bill-creating call whose reply was lost.
- **Close** a pending entry by publishing it at the next `v` in state `done`, then deleting the old wrap as for any change; never by deletion alone. A wallet MUST NOT remove the entries of a call's source bill(s), with any `secret`, until the pending entry is `done`.
- **Versions**: every write for a `w` uses the highest `v` seen for that `w` plus 1.
- **Change** an entry by publishing a new wrap with the next `v`, then deleting the previous wrap with a NIP-09 kind `5` signed by that wrap's `wrap_key`. The deletion's `created_at` is randomized like the wraps'. A wallet MAY batch deletions at a random later time.
- **Remove** an entry by deleting its wrap. A wallet MAY first keep it for up to 30 days in a final state (`spent`, `expired`, `reclaimed`, `done`), so the user can see why a note left.
- **Expiration**: a wallet MAY set a NIP-40 `expiration` on the gift wrap: `expires_at` plus a grace period (e.g. 30 days), rounded up to the week plus a random number of days. It still deletes the wrap itself, since relays may ignore NIP-40.

## Reading

- **Full sync**: `{"kinds": [1059], "#p": ["<user-pubkey>"]}`, then unwrap and keep rumors of kind `9525` whose seal `pubkey` and rumor `pubkey` are both the user. Other wraps (DMs) are ignored. A wallet MUST discard any other `9525`: anyone can gift-wrap to the user, but only the user can sign the seal.
- **Incremental sync**: the same filter with `since` set to the last sync minus 2 days, since gift wraps carry randomized timestamps. Wraps already seen are skipped.
- **Resolving**: per `w`, a final state (`spent`, `expired`, `reclaimed`, `done`) wins only over entries whose `v` is less than or equal to its own; an entry with a higher `v`, written after a definitive `cash_status` result, supersedes it (a bill returned by an in-place transfer keeps its `w`). Otherwise the highest `v` wins; on a tie, the latest rumor `created_at`. On any conflict the wallet re-checks the bill with `cash_status`.
- **Older entries**: a wallet SHOULD fill a missing `expires_at` from `cash_status` and rewrite the entry. A reader MUST ignore an entry whose `state` it does not recognise, except for the derived-id check in §Writing.
- **Resync**: drop the local copy and run a full sync.

## Balance

The balance, per `chain`, is the sum of `amount` over entries in state `live`. A wallet refreshes states with `cash_status` (NIP-CASH §Cash Status), batched per Hub, and updates entries whose state changed:

| Result | State |
|--------|-------|
| the wallet's own spend succeeded | `spent` |
| a tombstone after `expires_at` | `expired` |
| `NOT_FOUND` on a live bill, for an item proof signed by the entry's `identity` key or, for a `cash` entry, an item made with its `secret` | `reclaimed` |
| `NOT_FOUND` for any other proof | `unknown`, as no answer |
| any other tombstone | `spent` |
| no answer | `unknown`, never `spent` |

A device MUST write a final state only from a definitive result (its own spend, a tombstone, `NOT_FOUND` for a proof signed by the entry's `identity` key or, for a `cash` entry, an item made with its `secret`), never from a timeout or a cached view. Where the Hub quotes it, a wallet SHOULD show `net_redeemable_millis` rather than `amount`.

## Sweeping and Spending

- A wallet SHOULD claim received cash into the user's own Hub promptly. A claim is fee-free only when the user's Hub runs on the minting node; otherwise the redeem fee applies, and a bill at or below the base fee has to be consolidated first. The entry then becomes `spent`. The Pocket holds cash until claimed; a held bill returns to its minting Hub at `expires_at`.
- A wallet MUST show `expires_at`, and SHOULD warn and claim or redeem before it.
- A wallet SHOULD transfer a received cash-mode bill to the user's pubkey, so no `secret` is stored.
- To pay, a wallet SHOULD spend held tokens first (`cash_transfer`, whole or split), then mint from the user's Hub. A split leaves a new token for the remainder, which gets its own entry.
- If two devices spend the same token, the Hub accepts the first; the other device sees the token `spent` on its next refresh.

## Security Considerations

- **Custody**: a held token stays in its Minter's custody. Until claimed, the minting Hub's owner can revoke it (NIP-CZ §Security Considerations).
- **Bearer secrets**: an entry with `secret` is spendable by anyone who reads it. A stored `secret` can be spent by any app allowed to decrypt the user's DMs.
- **Stale wraps**: a wallet that fails to delete an old wrap leaves a stale copy behind. Resolving still picks the current one.

## Privacy Considerations

Relays see gift wraps addressed to the user at randomized times, the same as DMs, and deletions signed by one-time keys at randomized times. The number of wraps still grows with activity; a wallet MAY batch changes to hide it. Refreshing states contacts each Hub, not relays.

A wrap carrying NIP-40 `expiration` is distinguishable from a DM and reveals roughly when its bill expires; a wallet that wants wraps indistinguishable omits it.
