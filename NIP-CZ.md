NIP-CZ
======

Cash Zaps
---------

`draft` `optional`

**Depends on**: [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-AZ](./NIP-AZ.md) (AltZap), [NIP-IC](./NIP-IC.md) (Identity Connection)

## Abstract

A Cash Zap is a zap paid with a NIP-CASH cash token instead of a Lightning invoice. The sender mints a token for the recipient and publishes it in a kind `9522` event tagging the zapped note. A recipient lists the chains, minters and relays they accept in a kind `10522` event.

## Motivation

1. **Zapping without Lightning**: a user holding cash zaps a note without paying an invoice.
2. **Per-note tracking**: one public event per zap, counted beside `9521` receipts.
3. **Recipients without a wallet**: a token bound to a ConnectionKey waits, until it expires, for its recipient to sign in and claim it.

## Terminology

- **Cash Zap**: a kind `9522` event carrying a cash token minted for its `p`.
- **cash token**: a NIP-CASH token, `satscash1...` for Bitcoin (NIP-CASH §The Cash Token).
- **Minter**: the Lightning node that signed the token's mint signature (NIP-CASH §Mint Provenance), as a 33-byte compressed pubkey in hex.
- **ConnectionKey**: `hex(SHA256("<platform>:<identifier>"))`, standing in for someone with no Nostr keypair yet. A Nostr identity MUST NOT be hashed into one.
- **Proxy Agent**: a bot or service that signs a Cash Zap for a platform user whose keys it does not hold.
- **Chain**: the coin behind the token, e.g. `bitcoin`. Cash Zaps work the same for any energy-backed coin NIP-CASH supports; examples use Bitcoin.

## Specification

### Event Kinds

| Kind | Name | Signed by | Range |
|------|------|-----------|-------|
| `9522` | Cash Zap | Sender, or a Proxy Agent | regular |
| `10522` | Cash Zap Info | Recipient | replaceable |

### Identity Tags

`p` (recipient) and `P` (sender):

```
["p", "<hex-pubkey-or-ConnectionKey>", "<platform>"]
```

If `[2]` is missing, empty or `nostr`, `[1]` is a Nostr pubkey. Otherwise `[2]` names a platform (`discord`, `email`, `x`, ...) and `[1]` is a ConnectionKey.

### Sending

1. If `p` is a pubkey, fetch the recipient's `10522`. If it lists `chain` tags, the sender MUST use one; if it lists `mint` tags for that chain, the sender MUST mint from one of them.
2. Call `mint_cash` (NIP-CASH §Minting Cash) with exactly one recipient and `amount_millis` equal to the zap amount:
   - pubkey `p`: `{"identity_type": "pubkey", "identity_value": "<p>"}`
   - ConnectionKey `p`: `{"identity_type": "connection_key", "identity_value": "<p>", "ia_pubkey": "<ia>"}`, with `<ia>` an Identity Authority the sender's Hub trusts, also put in the `ia` tag.

   The sender MUST NOT mint a `cash`-mode recipient: its secret is not in the token, so only the sender could redeem it. On a retry, the sender SHOULD send an `idempotency_key`. For a ConnectionKey `p`, the sender SHOULD use the longest `expiry` the Hub allows.
3. Publish to the recipient's `10522` relays (else their [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) read relays) and to the sender's write relays. For a ConnectionKey `p`, also publish to its NIP-IC identity connection's relays if known, and SHOULD notify the recipient on their platform with an `nevent`.

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
- `cash`: exactly one.
- `w`: exactly one, the token's wallet pubkey (TLV `0`), so every copy of a token can be found with `#w`.
- `ia`: exactly one if `p` is a ConnectionKey, else none.

**Optional tags**: `e`, `a`, `k` (at most one each; `e` and `a` MUST name the same event), `P` (at most one), `expires_at` (at most one, from the mint response, informational).

A `9522` MUST NOT carry a NIP-40 `expiration` tag: relays could drop it before the recipient sees it.

Without `P`, the signer is the sender. With `P`, the signer is a Proxy Agent acting for `P`, and implementations MUST NOT attribute the zap to `pubkey`. Nothing proves `P`, so a client SHOULD credit `P` only if it trusts the signer, and otherwise show the zap as sent via the signer.

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
    ["chain", "<other-chain>"],
    ["mint", "<bitcoin-minter-node-pubkey>", "bitcoin"],
    ["mint", "<bitcoin-backup-minter-node-pubkey>", "bitcoin"],
    ["mint", "<other-chain-minter-node-pubkey>", "<other-chain>"],
    ["relay", "wss://relay.example.com"]
  ],
  "pubkey": "<recipient-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

- `chain`: one per accepted chain; none means any chain.
- `mint`: `["mint", "<minter-node-pubkey>", "<chain>"]`, a minter the recipient trusts on that chain. Cash Zaps from listed minters are **verified**. A chain with no `mint` accepts Cash Zaps as unverified. A `mint` for an unaccepted chain is ignored.
- `relay`: where the recipient's wallet reads Cash Zaps, for all chains.

The example accepts two chains, with two trusted minters on Bitcoin and one on the other. A recipient with no `10522`, including every ConnectionKey, accepts any chain, unverified.

## Verification Algorithm

- **Client**: shows or counts a `9522`, using only the event, the token and the recipient's `10522`.
- **Recipient wallet**: decides whether the cash is the recipient's.
- **Relay**: MAY skip validation; if it validates, it applies the Client column.

Signature verification MUST come before trusting any tag.

| # | Rule | Client | Recipient wallet |
|---|------|--------|------------------|
| 1 | `id`/`sig` verify against `pubkey`. | MUST | MUST |
| 2 | Tags meet the cardinality in §Kind 9522 — Cash Zap, and `w` equals the token's TLV `0`. | MUST | MUST |
| 3 | `cash` decodes per NIP-CASH §Wire Format. | MUST | MUST |
| 4 | The token's prefix matches `chain` (`satscash` ↔ `bitcoin`). | MUST | MUST |
| 5 | The token's identity-required hint (TLV `3`), if present, is `1`. | MUST | MUST |
| 6 | The mint signature recovers a Minter (NIP-CASH §Mint Provenance), and its attested amount (TLV `6`) equals `amount`. | MUST | MUST |
| 7 | `created_at` is not earlier than the target event's. | SHOULD | — |
| 8 | The Minter is in the recipient's `10522` `mint` tags for `chain`. | to count it | MUST, unless the recipient approves it |
| 9 | The recipient's slice is unclaimed and its `amount_millis` equals `amount`. For a pubkey `p`, `cash_status` with `scope: mine` (NIP-CASH §Cash Status) returns it; for a ConnectionKey `p`, a full in-place `cash_transfer` to the recipient's pubkey, using the attestation from `ia`, succeeds first. | — | MUST |

Rules 1–8 only show that a trusted node minted the amount. Only rule 9 proves the cash is the recipient's: a wallet's funding equals the sum of its slices, so a matching amount means nobody else shares it. TLV `1`–`3` are unsigned, so rule 5 only catches mistakes. Clients MUST NOT present a `9522` as value received.

## Receiving

- Incoming: `{"kinds": [9522], "#p": ["<recipient-pubkey>", "<connection-key>", ...]}` on the `10522` relays, including every ConnectionKey linked through NIP-IC. When linking a ConnectionKey, also query its identity connection's relays and any `nevent` received.
- Sent: `{"kinds": [9522], "authors": ["<user-pubkey>"]}` and `{"kinds": [9522], "#P": ["<user-pubkey>"]}`.
- A wallet MUST NOT contact a Minter missing from the recipient's `mint` tags without their approval, since contacting it reveals their identity and IP to that minter.
- A wallet MUST try the token from every copy sharing a `w` before calling it unclaimable: relay hints and the connection secret are unsigned.
- A token's state comes from `cash_status` only (NIP-CASH §Answering About a Destroyed Bill). A timeout MUST NOT be reported as spent.
- A wallet SHOULD keep the tokens it holds in a [NIP-CP](./NIP-CP.md) Cash Pocket.
- A wallet SHOULD claim promptly, since the sender can revoke until then. An external redemption pays `net_redeemable_millis`; a zap at or below the Hub's base fee can only be consolidated or redeemed same-node (NIP-CASH §The Redeem Fee).
- Clients SHOULD NOT publish when a Cash Zap was claimed.

## Counting

A client MAY total **verified** Cash Zaps (rules 1–8) per note, shown apart from Lightning zaps and as promised, not received. Unverified ones MAY be listed, never totalled.

A token counts once. If all copies of a `w` agree on target, `p` and sender, the earliest counts; if they disagree, none does.

A client MAY mark a Cash Zap past `expires_at` as possibly expired. A sender's NIP-09 deletion stops it counting, but does not revoke the token.

## Security Considerations

- **Fake value**: any key can make a valid mint signature, and a sender can mint to themselves and reclaim. Counting only recipient-listed minters limits this to minters the recipient trusts, as NIP-57 does with `nostrPubkey`.
- **Revocation**: until claimed, the sender can delete the bill, remove the slice, or untrust a ConnectionKey's IA (NIP-CASH §Lifecycle and Deletion, §Removing a Recipient).
- **Copies**: a copied token cannot be redeemed, but can be re-tagged to another note, recipient or sender, with any `created_at`. Conflicting copies are therefore not counted.
- **Split bills**: TLV `6` attests the whole wallet, so a bill could give the recipient one sat and the sender the rest. Only rule 9 catches it.
- **Proxy trust**: anyone can sign a `P` naming anyone.

## Privacy Considerations

A Cash Zap shows its amount, chain, recipient and Minter; for a self-hosted Hub, the Minter is the sender's own node.

Anyone holding the public token can poll `get_balance` and `cash_status` and learn when the value left — redeemed, reclaimed or expired, without telling which — exactly, via the tombstone's `retained_until`. A sender who wants this hidden SHOULD send the token privately instead.

## Future Work

Optional NIP-CASH upgrades, needing no change to these events:

- a mint-signature commitment to the recipient and target, making attribution provable and the copy rules unnecessary;
- irrevocable bills, turning counts from promised into committed;
- an attestation parameter on `cash_status`, so a ConnectionKey recipient can check rule 9 without a transfer.
