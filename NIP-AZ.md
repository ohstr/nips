NIP-AZ
======

AltZap: Cross-Chain Lightning Zaps
-----------------------------------

`draft` `optional`

**Depends on**: [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) (Lightning Zaps)

## Abstract

This NIP defines **AltZap**, a three-kind extension of NIP-57's zap request/receipt/LNURL flow (kinds `9520`, `9521`, `9523`). It generalizes NIP-57 along two axes NIP-57 does not address: settlement extends to any energy-backed coin routable over the Lightning Network, not just Bitcoin, and a recipient or sender may be identified by a deterministic **ConnectionKey** rather than requiring a Nostr keypair up front. AltZap is not wire-compatible with NIP-57 and does not claim to be — it defines its own event kinds specifically so aware and unaware clients cannot silently misinterpret each other's events.

## Motivation

NIP-57 assumes three things that don't hold for a payments network bridging non-Nostr identities across multiple chains:

1. **Single-chain settlement.** NIP-57 has no field asserting which network a zap settles on. A client or provider that supports more than one Lightning-routable chain has no way to prevent a request meant for one chain from being paid on another.
2. **Sender/recipient are always Nostr pubkeys.** A large class of useful recipients — a Discord user, an email address, an X handle — do not have a Nostr keypair at the time they're zapped. NIP-57's `p` tag has no room to say "this identifier isn't a pubkey yet."
3. **The signer is always the economic sender.** When a custodial bot generates a zap on behalf of a platform user it doesn't control the keys for, NIP-57 has no way to distinguish "the bot paid this" from "the bot is relaying a payment from someone else."

AltZap solves all three by adding a mandatory `chain` tag (replay-safety across chains), extending the `p`/`P` identity tags to a 3-element `["p/P", "<hex-or-ConnectionKey>", "<platform>"]` form, and introducing a dedicated on-behalf kind (`9523`) where the signing pubkey is explicitly *not* the party to trust for identity.

## Terminology

- **LNURL server**: The recipient's LNURL-pay server (NIP-57's "lnurl server"; LUD-06's "LN SERVICE"). It receives `9520`/`9523` requests at its callback, issues the invoices and publishes the resulting `9521` receipts.
- **Web Identity**: A non-Nostr identity platform (Discord, email, X). Nostr itself is **not** a Web Identity.
- **ConnectionKey**: A deterministic `SHA256("<platform>:<identifier>")` hash standing in for a Nostr pubkey when the real recipient/sender has no keypair yet. Native Nostr identities MUST NOT be hashed into a ConnectionKey — a real pubkey is used as-is.
- **Proxy Agent**: A bot or service that signs a `9523` event on behalf of a platform user it does not hold private keys for.
- **Chain**: The Lightning-routable settlement network, e.g. `flokicoin` or `bitcoin`.

## Specification

### Event Kinds

| Kind | Name | Signed by | Mechanism |
|------|------|-----------|-----------|
| `9520` | AltZap Request | Sender | LNURL-Pay |
| `9523` | On-Behalf AltZap Request | Proxy Agent | LNURL-Pay |
| `9521` | AltZap Receipt | LNURL server | — |

### LNURL Chain Advertisement

AltZap's multi-chain support has an LNURL-side counterpart to the event-level `chain` tag: a client needs to know which chains an endpoint accepts *before* it builds and signs a request. This is signaled through the standard LUD-06/LUD-16 `metadata` array, extended with `chain/<name>` entries — a backward-compatible addition existing Bitcoin-only wallets simply ignore.

```json
{
  "callback": "https://example.com/lnurl/callback/<id>",
  "maxSendable": 100000000,
  "minSendable": 1000,
  "metadata": "[[\"text/plain\",\"...\"],[\"chain/flokicoin\",\"loki\"]]",
  "tag": "payRequest",
  "allowsNostr": true,
  "nostrPubkey": "<lnurl-server-pubkey>"
}
```

- `allowsNostr: true` signals, as in NIP-57, that the callback accepts a signed Nostr event in its `nostr` query parameter. `nostrPubkey` is the pubkey that will sign the resulting receipt.
- Each `chain/<name>` entry in `metadata` advertises one supported chain; only the listed chains are supported. An endpoint with no `chain/<name>` entry settles on Bitcoin only and is a plain NIP-57 endpoint.
- An endpoint supports AltZap only if it has `allowsNostr: true` **and** at least one `chain/<name>` entry. A client MUST NOT send a `9520`/`9523` request to any other endpoint; it uses a NIP-57 `9734` request there instead.
- Amounts (`minSendable`/`maxSendable`, and the request's own `amount` tag) are denominated in that chain's smallest sub-unit: `mloki` for `chain/flokicoin`, `msats` for `chain/bitcoin`.
- A wallet that doesn't recognize `chain/flokicoin` simply attempts to pay the returned invoice and fails at the Lightning protocol level — the invoice belongs to a different network's node graph, so there is no route to it. No funds are ever at risk of landing on the wrong chain.

### LNURL Callback

As in NIP-57, the client sends the signed `9520`/`9523` event to the `callback` URL as a GET request with two query parameters:

- `amount`: the amount in milli-units; it MUST equal the event's `amount` tag.
- `nostr`: the event's JSON, URL-encoded.

The endpoint tells the two protocols apart by the event's `kind`: `9734` is handled as NIP-57, `9520`/`9523` as AltZap. It MUST reject a request whose `chain` tag is not one of the chains it advertises.

### The 3-Element Identity Tag

All `p` (recipient) and `P` (sender) tags use:

```
["p/P", "<hex-pubkey-or-ConnectionKey>", "<platform>"]
```

- `[1]` is a 64-character hex Nostr pubkey **or** a ConnectionKey.
- `[2]` is the platform name (`discord`, `email`, `x`, ...). If omitted or empty, it MUST be interpreted as `nostr` — `[1]` is a native pubkey.

`9521` receipts additionally allow a 4th element, `["p/P", "<hex-or-ConnectionKey>", "<platform>", "<handle>"]` — a stable, human-readable provider handle (e.g. a Discord username). It is informational only and MUST NOT be used for identity resolution.

### Kind 9520 — AltZap Request

```json
{
  "kind": 9520,
  "content": "Here is 1000 loki! Great post.",
  "tags": [
    ["e", "<optional-target-event-id>"],
    ["a", "<optional-kind:pubkey:d_tag>"],
    ["k", "<optional-target-event-kind>"],
    ["p", "<recipient-pubkey-or-ConnectionKey>", "<optional-platform>"],
    ["amount", "1000000"],
    ["chain", "flokicoin"],
    ["lnurl", "lnurl1..."],
    ["relays", "wss://relay.example.com"]
  ],
  "pubkey": "<sender-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

Signed by the sender's own key; the sender is always the event signer for `9520`.

**Required tags**: `p` (exactly one), `amount` (positive integer, milli-units — `mloki` or `msats`), `chain`, `lnurl` (bech32-encoded), `relays` (one or more `wss://`/`ws://` URLs).
**Optional tags**: `e` (at most one, 32-byte hex — the specific note being zapped), `a` (`<kind>:<pubkey>:<d-tag>` coordinate, for zapping a parameterized replaceable event), `k` (stringified kind number of the target event). A `9520` MUST NOT carry a `P` tag: the signer is the sender.

**Description-hash binding.** Once validated, the LNURL server issues a BOLT11 invoice whose `description_hash` equals the SHA256 of the event JSON exactly as received in the `nostr` parameter (after URL-decoding). The LNURL server MUST NOT re-serialize the event before hashing, and the receipt's `description` tag carries that same string verbatim. This is what lets a `9521` receipt's `description` tag be checked against the invoice that was actually paid (see Verification Algorithm below) — the invoice is cryptographically bound to this exact request, not just to an amount.

### Kind 9523 — On-Behalf AltZap Request

Identical structure to `9520`, kind `9523` instead, with one additional constraint: **exactly one `P` (sender) tag is REQUIRED**. The event's own `pubkey` field (the Proxy Agent's signing key) MUST NOT be treated as the economic sender — implementations MUST read `P` instead.

```json
{
  "kind": 9523,
  "content": "Sent via Proxy Application",
  "tags": [
    ["p", "<recipient-ConnectionKey>", "<platform>"],
    ["P", "<sender-ConnectionKey>", "<platform>"],
    ["amount", "100000"],
    ["chain", "flokicoin"],
    ["relays", "wss://relay.example.com"],
    ["lnurl", "lnurl1..."]
  ],
  "pubkey": "<proxy-agent-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

### Kind 9521 — AltZap Receipt

```json
{
  "kind": 9521,
  "content": "",
  "tags": [
    ["e", "<optional-target-event-id>"],
    ["a", "<optional-kind:pubkey:d_tag>"],
    ["p", "<recipient-hex-or-ConnectionKey>", "<platform>", "<handle>"],
    ["P", "<sender-hex-or-ConnectionKey>", "<platform>", "<handle>"],
    ["r", "<resolved-recipient-pubkey>"],
    ["R", "<resolved-sender-pubkey>"],
    ["amount", "<settled-milli-units>"],
    ["chain", "flokicoin"],
    ["bolt11", "lnfc10u1..."],
    ["description", "<json-string-of-the-original-request>"],
    ["preimage", "<32-byte-hex>"]
  ],
  "pubkey": "<lnurl-server-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

Signed by the LNURL server once the invoice settles, published to the relays named in the originating request's `relays` tag.

**Required tags**: `bolt11`, `chain`, `preimage`. `p` is required whenever `description` is present.
**Optional tags**: `e`/`a` (mirrored from the request), `amount` (the settled amount in milli-units, informational), `description` (the stringified original `9520`/`9523` event — omitted for a plain LNURL payment settled without any signed request), `r`/`R` (resolved native pubkeys, see below).

#### `r` / `R` — Resolved Identity

When the LNURL server can independently map a ConnectionKey back to a real Nostr pubkey (e.g. via a `kind:35521` identity connection it has indexed), it MAY attach that pubkey as `r` (for a ConnectionKey `p`) or `R` (for a ConnectionKey `P`). This lets a client render a "Social Bridge" experience — showing the real recipient's profile — without waiting on the recipient to claim funds.

#### Identity Tag Matrix

| Intent kind | Sender (`P`) | Recipient (`p`) | `r` | `R` |
|---|---|---|---|---|
| `9520` | Signer pubkey (no `P` on the request; the receipt sets it) | Pubkey or ConnectionKey | If `p` is a ConnectionKey and resolved | N/A — sender is already a pubkey |
| `9523` | ConnectionKey | ConnectionKey | If resolved | If resolved |

## Verification Algorithm

Implementations MUST enforce all of the following when validating an inbound `9520`/`9523` request. Signature verification MUST happen before any tag content is trusted; the remaining checks are independent and order does not otherwise matter.

1. **Signature.** The event's `id`/`sig` MUST verify against `pubkey` per NIP-01.
2. **Identity tags.** MUST have exactly one `p` tag and a non-empty `lnurl` tag.
3. **`chain`.** MUST be present and non-empty. An LNURL server that only routes a subset of chains MUST reject requests naming an unsupported chain.
4. **`amount`.** MUST parse as a positive integer. If the caller has an expected amount in hand (e.g. from the LNURL callback's own `amount` query parameter), it MUST equal the tag value exactly.
5. **`relays`.** MUST contain at least one `ws://`/`wss://` URL.
6. **`e`.** At most one is allowed; if present, MUST be a 32-byte hex ID.
7. **`9523`-specific.** MUST have exactly one `P` tag. A `9520` MUST NOT have a `P` tag.

For a `9521` receipt:

1. **Signature**, as above.
2. **`chain`, `preimage`, `bolt11`** MUST be present.
3. **Hash lock.** `SHA256(preimage)` MUST equal the invoice's payment hash.
4. **Chain.** The `bolt11` invoice MUST belong to the network named in the receipt's `chain` tag.
5. If `description` is present, it MUST parse as valid JSON, embed a validly-signed request event, and: `SHA256(description)` MUST equal the invoice's `description_hash`; the invoice's settled amount MUST equal the embedded request's `amount`; the receipt's `p` MUST equal the embedded request's `p`; the receipt's `chain` MUST equal the embedded request's `chain`.
6. If `description` is absent (a plain LNURL payment settled without a signed request), the invoice's settled amount MUST be positive — there is no request to cross-check against.
7. If `amount` is present, it MUST equal the invoice's settled amount.

## Custodial Settlement

An LNURL server MAY act as a temporary custodian when the resolved recipient has no reachable wallet at settlement time — either because `p` is a ConnectionKey with no linked identity yet, or because a linked Nostr wallet is unresponsive. In both cases the LNURL server publishes the `9521` receipt as normal and holds the settled value internally against the recipient's ConnectionKey or pubkey until they authenticate and connect a wallet, at which point the LNURL server sweeps the balance to them. This is an operational behavior of the LNURL server, not a protocol-level tag or event.

## Backwards Compatibility

AltZap is intentionally **not** wire-compatible with NIP-57: a client that only understands `9734`/`9735` will neither generate nor recognize `9520`–`9523` events, and vice versa. This is deliberate — silently mixing the two wire formats under shared kinds would let a chain-unaware client misinterpret a chain-gated request. An LNURL server that wants to serve both ecosystems runs both flows side by side, keyed on kind number.

## Security Considerations

- **Cross-chain replay**: the mandatory `chain` tag exists specifically so a request signed for one chain cannot be settled — accidentally or maliciously — against a different chain's liquidity. Reject on chain mismatch; do not attempt "best effort" chain substitution.
- **Proxy trust boundary**: `9523`'s signing `pubkey` is the Proxy Agent, not the sender. Clients and LNURL servers MUST read `P`, not `pubkey`, when displaying or attributing the sender — treating the signer as the sender defeats the entire purpose of the kind.
- **ConnectionKey collisions**: a ConnectionKey is a one-way hash of `"<platform>:<identifier>"`. Implementations MUST NOT accept a claimed reverse mapping without independent verification (e.g. a signed identity-connection event) — the receipt alone is not proof of ownership.
- **`r`/`R` disclosure**: resolving a ConnectionKey to a real pubkey in a receipt is a deliberate, visible linkage of a Web Identity to a Nostr identity. An LNURL server SHOULD only populate `r`/`R` when it has verified authorization to make that link public.

## Relation to Other NIPs

**NIP-57**: AltZap extends NIP-57's conceptual request/receipt/LNURL flow but is a distinct, non-interoperable wire format (see Backwards Compatibility).

**[NIP-IC](./NIP-IC.md)** (Identity Connection, `kind:35521`/`kind:35522`): AltZap's ConnectionKey mechanism is protocol-agnostic about *how* a key gets resolved to a pubkey — resolving `r`/`R` via a NIP-IC identity connection is one way to do it, not a requirement of this spec.
