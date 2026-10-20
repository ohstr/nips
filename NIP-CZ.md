NIP-CZ
======

Cash Zaps
---------

`draft` `optional`

**Depends on**: [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md), [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-AZ](./NIP-AZ.md) (AltZap) — the Lightning counterpart, sharing the `chain` tag, the 3-element `p`/`P` tag and the ConnectionKey; [NIP-IC](./NIP-IC.md) (Identity Connection) — links a ConnectionKey recipient to a pubkey.

## Abstract

This NIP defines **Cash Zaps**: a zap paid with a NIP-CASH cash token instead of a Lightning invoice. The sender mints a token for one recipient and publishes it in a kind `9522` event that tags the zapped note. The token carries the minting node's signature over its amount, so there is no LNURL server, no invoice and no separate receipt. A recipient names the chains, minters and relays they accept in a replaceable kind `10522` event.

## Motivation

1. **Zapping without Lightning.** A user holding cash wants to zap a note without paying an invoice. NIP-AZ settles only through LNURL.
2. **Per-event tracking.** A Cash Zap is one public event per zap, tagged to its target, so clients count it beside `9521` receipts and wallets find incoming cash with one filter.
3. **Recipients without a wallet.** A token bound to a ConnectionKey waits in the sender's Cash Hub, until its expiry, for its recipient to sign in and claim it. No LNURL server holds funds on their behalf (compare NIP-AZ §Custodial Settlement).

## Terminology

- **Cash Zap**: a kind `9522` event carrying a cash token minted for its `p`.
- **cash token**: a NIP-CASH bech32 token, `satscash1...` for Bitcoin (NIP-CASH §The Cash Token).
- **Minter**: the Lightning node whose identity key produced the token's mint signature (NIP-CASH §Mint Provenance), named by its 33-byte compressed pubkey in hex.
- **ConnectionKey**: `hex(SHA256("<platform>:<identifier>"))`, standing in for the pubkey of a user who has no Nostr keypair yet. A native Nostr identity MUST NOT be hashed into one.
- **Proxy Agent**: a bot or service that signs a Cash Zap on behalf of a platform user whose keys it does not hold.
- **Chain**: the energy-backed coin behind the token, e.g. `bitcoin`. Cash Zaps work the same for any energy-backed coin NIP-CASH extends to; examples here use Bitcoin.

## Specification

### Event Kinds

| Kind | Name | Signed by | Range |
|------|------|-----------|-------|
| `9522` | Cash Zap | Sender, or a Proxy Agent | regular |
| `10522` | Cash Zap Info | Recipient | replaceable |

### Identity Tags

`p` (recipient) and `P` (sender) use NIP-AZ's form:

```
["p", "<hex-pubkey-or-ConnectionKey>", "<platform>"]
```

If `[2]` is missing, empty or `nostr`, `[1]` is a Nostr pubkey. Any other value names a Web Identity platform (`discord`, `email`, `x`, ...) and `[1]` is a ConnectionKey for it.

### Sending

1. If `p` is a pubkey, fetch the recipient's `10522`, if any. If it lists `chain` tags, the sender MUST use one of them; if it lists `mint` tags for that chain, the sender MUST mint from one of those minters.
2. Call `mint_cash` (NIP-CASH §Minting Cash) with **exactly one** recipient and `amount_millis` equal to the zap amount:
   - a pubkey `p` → `{"identity_type": "pubkey", "identity_value": "<p>"}`;
   - a ConnectionKey `p` → `{"identity_type": "connection_key", "identity_value": "<p>", "ia_pubkey": "<ia>"}`, where `<ia>` is an Identity Authority on the sender's Hub allowlist. It goes in the event's `ia` tag.

   The sender MUST NOT mint a `cash`-mode recipient: its `cash_secret` is not in the token (NIP-CASH §Redemption Metadata), so only the sender could ever redeem it. A sender retrying a timed-out call SHOULD send an `idempotency_key`. For a ConnectionKey `p`, the sender SHOULD set the longest `expiry` the Hub's ceiling allows, or omit it to get that ceiling.
3. Publish the `9522` to the recipient's `10522` relays (falling back to their [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) read relays) and to the sender's own write relays. For a ConnectionKey `p`, also publish to the relays of the recipient's NIP-IC identity connection if one is known, and SHOULD notify them on their platform with a NIP-19 `nevent`.

### Kind 9522 — Cash Zap

```json
{
  "kind": 9522,
  "content": "Great post!",
  "tags": [
    ["e", "<optional-target-event-id>"],
    ["k", "<optional-target-event-kind>"],
    ["p", "<recipient-pubkey>"],
    ["amount", "21000"],
    ["chain", "bitcoin"],
    ["cash", "satscash1..."],
    ["w", "<token-wallet-pubkey>"]
  ],
  "pubkey": "<sender-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

**Required tags**:
- `p`: exactly one.
- `amount`: exactly one, a positive integer in the chain's milli-unit (`msats` for `bitcoin`).
- `chain`: exactly one.
- `cash`: exactly one cash token.
- `w`: exactly one, the token's wallet pubkey (TLV `0`) in hex, so every copy of a token can be fetched with `#w`.
- `ia`: exactly one when `p` is a ConnectionKey, the `ia_pubkey` it was minted with; absent otherwise.

**Optional tags**: `e`, `a` and `k`, at most one each (when `e` and `a` are both present they MUST name the same event); `P`, at most one; `expires_at`, at most one, the mint response's `expires_at` in unix seconds.

`expires_at` is set by the sender and is informational only. A `9522` MUST NOT carry a NIP-40 `expiration` tag, which would let relays drop it before its recipient sees it.

Without `P`, the signer is the sender. With `P`, the signer is a Proxy Agent acting for `P`, as on NIP-AZ's `9523` but on the same kind. Implementations MUST NOT attribute the zap to `pubkey`. Nothing proves `P`, so a client SHOULD attribute it to `P` only when it trusts the signer, and otherwise show it as sent via the signer.

```json
{
  "kind": 9522,
  "content": "Sent via Proxy Application",
  "tags": [
    ["p", "<recipient-ConnectionKey>", "discord"],
    ["P", "<sender-ConnectionKey>", "discord"],
    ["amount", "100000"],
    ["chain", "bitcoin"],
    ["cash", "satscash1..."],
    ["w", "<token-wallet-pubkey>"],
    ["ia", "<ia-pubkey>"],
    ["expires_at", "1790000000"]
  ],
  "pubkey": "<proxy-agent-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

### Kind 10522 — Cash Zap Info

```json
{
  "kind": 10522,
  "content": "",
  "tags": [
    ["chain", "bitcoin"],
    ["mint", "<minter-node-pubkey>", "bitcoin"],
    ["relay", "wss://relay.example.com"]
  ],
  "pubkey": "<recipient-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

- `chain`: one per accepted chain. None means any chain.
- `mint`: `["mint", "<minter-node-pubkey>", "<chain>"]`, zero or more — the minters the recipient trusts. Cash Zaps from a listed minter are **verified** (§Counting). With no `mint` for a chain, Cash Zaps on it are accepted but unverified. A `mint` for a chain not accepted is ignored.
- `relay`: one or more relays the recipient's wallet reads Cash Zaps from.

A recipient with no `10522`, including every ConnectionKey, accepts any chain, and all their Cash Zaps are unverified.

## Verification Algorithm

- **Client**: displays or counts a `9522`. Checks what the event, the token and the recipient's `10522` prove without contacting a Hub.
- **Recipient wallet**: decides whether the cash is its user's.
- **Relay**: MAY skip validation entirely; one that validates applies the Client column.

Signature verification MUST happen before any tag content is trusted.

| # | Rule | Client | Recipient wallet |
|---|------|--------|------------------|
| 1 | `id`/`sig` verify against `pubkey` per NIP-01. | MUST | MUST |
| 2 | Tags meet the cardinality in §Kind 9522 — Cash Zap, and `w` equals the token's TLV `0`. | MUST | MUST |
| 3 | `cash` decodes per NIP-CASH §Wire Format. | MUST | MUST |
| 4 | The token's HRP names the coin in `chain` (`satscash` ↔ `bitcoin`). | MUST | MUST |
| 5 | The token's identity-required hint (TLV `3`), if present, is `1`. | MUST | MUST |
| 6 | The mint signature recovers a Minter (NIP-CASH §Mint Provenance), and the attested amount (TLV `6`) equals `amount`. | MUST | MUST |
| 7 | `created_at` is not earlier than the target event's. | SHOULD | — |
| 8 | The Minter is in the recipient's `10522` `mint` tags for `chain`. | to count it | MUST, unless the user approves the minter |
| 9 | The user's slice is unclaimed and its `amount_millis` equals `amount`: for a pubkey `p`, `cash_status` with `scope: mine` (NIP-CASH §Cash Status) returns that row; for a ConnectionKey `p`, a full in-place `cash_transfer` to the user's own pubkey with the attestation from `ia` succeeds first. | — | MUST |

Rules 1–8 prove that the event is well formed and that a trusted node minted the amount. The token names neither its recipient nor its event, and TLV `1`–`3` are unsigned, so rule 5 catches only honest mistakes. Only rule 9 proves the cash is the user's: since a wallet's funding equals the sum of its slices, a matching `amount_millis` also proves nobody else holds part of it. Clients MUST NOT present a `9522` as value received.

## Receiving

- A wallet finds incoming cash with `{"kinds": [9522], "#p": ["<user>", "<connection-key>", ...]}` on its `10522` relays, listing every ConnectionKey its user has linked through NIP-IC. When onboarding a ConnectionKey, it also queries that identity connection's relays and resolves any `nevent` it was sent.
- It finds sent history with `{"kinds": [9522], "authors": ["<user>"]}` and `{"kinds": [9522], "#P": ["<user>"]}`.
- A wallet MUST NOT contact a Minter outside its user's `mint` tags without the user's approval: doing so reveals the user's identity and IP to whoever minted the token.
- Copies of one token share `w`. Relay hints and the connection secret are unsigned, so a wallet MUST try the token from every copy before reporting it unclaimable.
- Live, spent or unknown comes from `cash_status` only (NIP-CASH §Answering About a Destroyed Bill). A wallet MUST NOT report a token spent on a timeout alone.
- A wallet SHOULD claim promptly: until then the sender can revoke (§Security Considerations). An external redemption pays `net_redeemable_millis`, `amount` minus the slice's redeem fee; a zap at or below the Hub's base fee can only be consolidated or redeemed same-node (NIP-CASH §The Redeem Fee).
- This NIP defines no redemption-status event, and a client SHOULD NOT publish one: it would reveal when the recipient claimed.

## Counting

A client MAY total Cash Zaps per note beside `9521` receipts. It counts only **verified** Cash Zaps, those passing rules 1–8, and SHOULD show them separately from Lightning zaps and as promised, not received. Unverified ones MAY be listed, never totalled.

Each token counts at most once. If every copy of a `w` agrees on target, `p` and sender, the earliest by `created_at` is counted. If copies disagree on any of them, none is counted.

A client MAY mark a Cash Zap past its `expires_at` as possibly expired. A sender's NIP-09 deletion withdraws its attribution: clients stop counting that event, but the token is not revoked, and wallets keep the tokens they have found.

## Security Considerations

- **Fake value**: any key can produce a valid mint signature, and a sender can mint to themselves and reclaim it. Counting only recipient-listed minters bounds this to minters the recipient trusts, as NIP-57 bounds receipts to the recipient's `nostrPubkey`. It does not prove the funds exist (NIP-CASH §Mint Provenance).
- **Revocation**: until a Cash Zap is claimed, its sender, as the bill's owner, can delete the bill, remove the slice, or drop a ConnectionKey's IA from their allowlist (NIP-CASH §Lifecycle and Deletion, §Removing a Recipient). A count is a promise.
- **Copies**: copying a token into another `9522` cannot redeem it, but can claim it for another note, recipient or sender. `created_at` is the copier's to set, so conflicting copies are not counted at all. A copy may also carry broken relay hints or secret, which is why wallets try every copy.
- **Split bills**: TLV `6` attests the whole wallet, so a bill could pay the recipient one sat and the sender the rest. Rule 9's amount check catches it; clients cannot.
- **Proxy trust**: anyone can sign a `P` naming anyone, so `P` is only as trustworthy as its signer.

## Privacy Considerations

A Cash Zap publishes its amount, chain, recipient and Minter. For a self-hosted Hub, the Minter is the sender's own Lightning node, linking it to their pubkey.

The token is public, so anyone can poll it: `get_balance` over NWC, and `cash_status`, which answers `NOT_FOUND` while the bill is live and a tombstone after. That reveals when the value left the bill — redeemed, reclaimed or expired, indistinguishably — exactly, given the tombstone's `retained_until` (NIP-CASH §Answering About a Destroyed Bill). A sender who needs any of this hidden SHOULD hand the token over privately instead.

## Future Work

NIP-CASH could make Cash Zaps stronger without changing this NIP's events:

- a mint-signature commitment to the recipient and a caller-chosen value (e.g. `sha256(sender ‖ p ‖ target)`) would make addressing and attribution provable offline, retiring the copy rules;
- an irrevocable-bill flag, attested by the mint signature, would turn counts from promised into committed;
- an attestation parameter on `cash_status` would let a ConnectionKey recipient check rule 9 without a transfer.

## Relation to Other NIPs

**[NIP-CASH](./NIP-CASH.md)** (Cash Hub): defines the token, `mint_cash`, `cash_status`, `cash_transfer` and redemption. This NIP only defines how a token is published as a zap.

**[NIP-AZ](./NIP-AZ.md)** (AltZap, `kind:9520`/`kind:9521`/`kind:9523`): the Lightning counterpart. Cash Zaps share its `chain` tag, 3-element `p`/`P` tag and ConnectionKey; `P` marks a Proxy Agent signer as on `9523`, but on the same kind.

**[NIP-IC](./NIP-IC.md)** (Identity Connection, `kind:35521`/`kind:35522`): links a ConnectionKey to its recipient's pubkey and supplies the attestation from `ia`.

**[NIP-61](https://github.com/nostr-protocol/nips/blob/master/61.md)** (Nutzaps, `kind:9321`/`kind:10019`): the same shape for Cashu ecash — tokens in a public event, plus a recipient info event.
