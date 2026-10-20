NIP-CZ
======

Cash Zaps
---------

`draft` `optional`

**Depends on**: [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md), [NIP-CASH](./NIP-CASH.md) (Cash Hub)
**Related**: [NIP-AZ](./NIP-AZ.md) (AltZap) — shares the `chain` tag, the 3-element `p`/`P` tag and the ConnectionKey; [NIP-IC](./NIP-IC.md) — resolves a ConnectionKey recipient to a pubkey.

## Abstract

This NIP defines **Cash Zaps**: a zap paid in NIP-CASH cash instead of a Lightning invoice. The sender mints a cash token bound to one recipient and publishes it in a kind `9522` event that tags the zapped note. The token carries its own proof of value — the minting node's signature over its amount — so there is no LNURL server, no invoice and no separate receipt. A recipient advertises where and from whom they accept Cash Zaps with a replaceable kind `10522` event.

## Motivation

1. **Zapping without Lightning.** A user holding cash wants to zap a note from a feed without paying an invoice. NIP-AZ settles only through LNURL, and NIP-CASH says a token can ride in a zap without defining how.
2. **Per-event tracking.** A Cash Zap is one public event per zap, tagged to its target, so feeds count it the way they count `9521` receipts, and a wallet indexes it to find cash sent to its user.
3. **Recipients without a wallet.** A token bound to a ConnectionKey waits, until its expiry, for its recipient to sign in and redeem it. No server has to hold the funds on their behalf (compare NIP-AZ §Custodial Settlement).

## Terminology

- **Cash Zap**: a kind `9522` event carrying a cash token minted for its `p`.
- **cash token**: a NIP-CASH bech32 token (`satscash1...` for Bitcoin, and a coin-specific prefix for any other energy-backed coin). See NIP-CASH §The Cash Token.
- **Minter**: the Lightning node whose identity key signed the token's mint signature (NIP-CASH §Mint Provenance), identified by its 33-byte compressed pubkey in hex.
- **ConnectionKey**: `SHA256("<platform>:<identifier>")`, standing in for the pubkey of a recipient or sender who has no Nostr keypair yet, as in NIP-AZ.
- **Proxy Agent**: a bot or service that signs a Cash Zap on behalf of a platform user whose keys it does not hold, as in NIP-AZ.
- **Chain**: the energy-backed coin backing the token, e.g. `bitcoin`. Cash Zaps work the same for any energy-backed coin NIP-CASH extends to; examples here use Bitcoin.

## Event Kinds

| Kind | Name | Signed by | Range |
|------|------|-----------|-------|
| `9522` | Cash Zap | Sender, or a Proxy Agent | regular |
| `10522` | Cash Zap Info | Recipient | replaceable |

## Sending

1. Read the recipient's `10522` (if `p` is a pubkey and one exists).
2. Call `mint_cash` (NIP-CASH §Minting Cash) with **exactly one** recipient and `amount_millis` equal to the zap amount:
   - a pubkey `p` → `{"identity_type": "pubkey", "identity_value": "<p>"}`;
   - a ConnectionKey `p` → `{"identity_type": "connection_key", "identity_value": "<p>", "ia_pubkey": "<ia>"}`, where `<ia>` is an Identity Authority the sender's Hub accepts; it goes in the event's `ia` tag.

   The sender MUST NOT mint a `cash`-mode recipient. Its `cash_secret` never appears in the token (NIP-CASH §Redemption Metadata), so nobody, the recipient included, could redeem it from the event.
3. If the recipient's `10522` lists `mint` tags for the chain, the sender SHOULD mint from one of those minters.
4. For a ConnectionKey `p`, the sender SHOULD set an `expiry` long enough for a recipient who has never used Nostr to sign in, or omit it for the Hub's ceiling.
5. Publish the `9522` to the recipient's `10522` relays, falling back to their [NIP-65](https://github.com/nostr-protocol/nips/blob/master/65.md) read relays, then to the sender's own. A ConnectionKey recipient has neither, so the sender (or its Proxy Agent) SHOULD also notify them on their platform with a NIP-19 `nevent` carrying relay hints.

## Kind 9522 — Cash Zap

```json
{
  "kind": 9522,
  "content": "Great post!",
  "tags": [
    ["e", "<optional-target-event-id>"],
    ["a", "<optional-kind:pubkey:d_tag>"],
    ["k", "<optional-target-event-kind>"],
    ["p", "<recipient-pubkey-or-ConnectionKey>", "<optional-platform>"],
    ["amount", "21000"],
    ["chain", "bitcoin"],
    ["cash", "satscash1..."],
    ["relays", "wss://relay.example.com"]
  ],
  "pubkey": "<sender-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

**Required tags**: `p` (exactly one, NIP-AZ's 3-element form), `amount` (exactly one, positive integer, the chain's milli-unit — `msats` for `bitcoin`), `chain` (exactly one), `cash` (exactly one cash token), `relays` (one or more `wss://`/`ws://` URLs), `ia` (exactly one when `p` is a ConnectionKey, the `ia_pubkey` it was minted with; absent otherwise).
**Optional tags**: `e` (at most one, the note being zapped), `a` (at most one, `<kind>:<pubkey>:<d-tag>` coordinate), `k` (at most one, stringified kind of the target), `P` (at most one, see below), `expires_at` (at most one, the mint response's `expires_at`, unix seconds; informational).

Without `P`, the signer is the sender. With `P`, the signer is a Proxy Agent and `P` is the sender: implementations MUST NOT attribute the zap to `pubkey`. Nothing in the event proves `P`, so a client SHOULD attribute it to `P` only when it trusts the signing Proxy Agent, and otherwise show the zap as sent via the signer.

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
    ["ia", "<ia-pubkey>"],
    ["expires_at", "1790000000"],
    ["relays", "wss://relay.example.com"]
  ],
  "pubkey": "<proxy-agent-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

## Kind 10522 — Cash Zap Info

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

- `chain`: one per accepted chain. None means any chain. Otherwise, a sender MUST NOT send a Cash Zap on a chain not listed.
- `mint`: `["mint", "<minter-node-pubkey>", "<chain>"]`, zero or more. None for a chain means any minter is accepted on it. A `mint` for a chain the event doesn't accept is ignored.
- `relay`: one or more relays the recipient's wallet reads Cash Zaps from.

A recipient with no `10522` (including every ConnectionKey) accepts any chain and any minter.

## Verification Algorithm

Two actors validate a `9522`:

- **Client / Relay**: displays or stores it. Checks only what the event and token prove offline.
- **Recipient wallet**: decides whether the cash is its user's. Enforces every rule.

Signature verification MUST happen before any tag content is trusted; the remaining checks are independent.

| # | Rule | Client / Relay | Recipient wallet |
|---|------|----------------|------------------|
| 1 | `id`/`sig` verify against `pubkey` per NIP-01. | MUST | MUST |
| 2 | Tag cardinality as listed above: exactly one `p`, `amount`, `chain` and `cash`; at least one `relays`; `ia` exactly when `p` is a ConnectionKey; at most one `P`, `e`, `a`, `k` and `expires_at`. | MUST | MUST |
| 3 | `cash` decodes per NIP-CASH §Wire Format. | MUST | MUST |
| 4 | The token's HRP names the coin in `chain` (`satscash` ↔ `bitcoin`). | MUST | MUST |
| 5 | The token's identity-required hint (TLV `3`) is present and `1`. | MUST | MUST |
| 6 | The mint signature recovers a Minter (NIP-CASH §Mint Provenance), and the attested amount (TLV `6`) equals `amount`. | MUST | MUST |
| 7 | The Minter is listed in the wallet's own `10522` for `chain`, if that lists any. | — | MUST |
| 8 | `cash_status` with `scope: mine` (NIP-CASH §Cash Status) returns the user's own row. | — | MUST |

Rules 1–6 prove only that the event is well formed and the token is real. The token names neither its recipient nor its event, and TLV `3` is a hint that can go stale (NIP-CASH §Redemption Metadata). Only rule 8 proves the cash is the recipient's. Clients therefore show value minted and addressed, never value received.

## Receiving and Indexing

- A wallet finds incoming cash with `{"kinds": [9522], "#p": ["<user>", "<connection-key>", ...]}` on its `10522` relays, listing every ConnectionKey its user has linked through NIP-IC, and SHOULD resolve any `nevent` it was sent on another platform.
- It finds its user's sent history with `{"kinds": [9522], "authors": ["<user>"]}` and `{"kinds": [9522], "#P": ["<user>"]}`.
- Duplicates — the same token in more than one event — are collapsed by the token's wallet pubkey (TLV `0`).
- Whether a token is live, spent or unknown comes from `cash_status` only (NIP-CASH §Answering About a Destroyed Bill). A wallet MUST NOT report a token spent on a timeout alone.
- Clients MUST NOT publish redemption status. How a wallet stores its own ledger is outside this NIP.

## Counting

A client MAY total Cash Zaps per event alongside `9521` receipts, counting only events that pass rules 1–6. It SHOULD show them as cash.

A token is not bound to its event: anyone can copy it into a new `9522` for another note or sender. A client MUST therefore count each token (TLV `0`) at most once across all events it has seen, keeping the earliest by `created_at`, ties broken by lowest `id`.

A Cash Zap can expire unredeemed and return to the sender's Cash Hub (NIP-CASH §Lifecycle and Deletion); its event stays published. A client MAY mark one past its `expires_at` as possibly expired.

## Security Considerations

- **Untrusted minters**: anyone can run a Cash Hub. A valid mint signature proves who minted a token, not that the funds still exist (NIP-CASH §Mint Provenance). Wallets SHOULD accept tokens only from minters their user trusts, and clients MAY weight counts the same way.
- **Cash-mode tokens**: a `cash`-mode slice is redeemed with a `cash_secret` the token doesn't carry, so one published in a `9522` is stranded until it expires. Rule 5 rejects a token that says so, or doesn't say.
- **Replay**: copying a token into another `9522` can't redeem it, since redemption needs the recipient's identity, but it can claim the zap for another note or sender. Counting's once-per-token rule caps the damage at one misattributed zap. Which copy is the original cannot be proven: `created_at` is the copier's to set.
- **Proxy trust boundary**: as in NIP-AZ, a `P`-tagged event's signer is not the sender. Anyone can sign a `P` naming anyone, so `P` is only as trustworthy as its signer.

## Privacy Considerations

A Cash Zap publishes its amount, chain, recipient and, through the mint signature, its Minter. Because the token is public, anyone can also ask the Hub about it after it is spent and receive the `spent` tombstone (NIP-CASH §Answering About a Destroyed Bill), learning roughly when it was redeemed. A sender who needs either hidden SHOULD hand the token over privately instead.

## Relation to Other NIPs

**[NIP-CASH](./NIP-CASH.md)**: defines the token, `mint_cash`, `cash_status` and redemption. This NIP only defines how a token is published as a zap.

**[NIP-AZ](./NIP-AZ.md)**: the Lightning counterpart. Cash Zaps reuse its `chain` tag, 3-element `p`/`P` tag, ConnectionKey and Proxy Agent semantics, and kind `9522` sits beside its `9520`, `9521` and `9523`.

**[NIP-IC](./NIP-IC.md)**: a ConnectionKey recipient proves their Web Identity through the Identity Authority named in `ia` before redeeming, and their linked ConnectionKeys are what a wallet adds to its `#p` filter.

**[NIP-61](https://github.com/nostr-protocol/nips/blob/master/61.md)** (Nutzaps): the same shape for Cashu ecash — locked tokens in a public event, plus a recipient info event.
