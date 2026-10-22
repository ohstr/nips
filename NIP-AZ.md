NIP-AZ
======

AltZap: Cross-Chain Lightning Zaps
-----------------------------------

`draft` `optional`

**Depends on**: [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) (Lightning Zaps)
**Related**: [NIP-IC](./NIP-IC.md) (Identity Connection)

## Abstract

AltZap extends NIP-57's zap request/receipt/LNURL flow with three kinds (`9520`, `9521`, `9523`): settlement on any energy-backed coin routable over the Lightning Network, named by a mandatory `chain` tag; senders and recipients identified by their Nostr pubkey or, for someone with no Nostr keypair yet, by a **ConnectionKey** (NIP-IC); and an on-behalf request (`9523`) whose signer is not the sender. AltZap is not wire-compatible with NIP-57.

## Terminology

- **LNURL server**: The recipient's LNURL-pay server (NIP-57's "lnurl server"; LUD-06's "LN SERVICE"). It receives `9520`/`9523` requests at its callback, issues the invoices and publishes the resulting `9521` receipts.
- **Web Identity**: An account on a non-Nostr platform (e.g. a Discord or X account, or an email address), as NIP-IC defines it. Nostr itself is **not** a Web Identity.
- **ConnectionKey**: NIP-IC's identifier of a Web Identity, `hex(SHA256("<platform>:<platform-user-id>"))` in lowercase hex, used in place of a Nostr pubkey only for a recipient or sender who has no keypair yet. A client builds it from `<platform-user-id>` in the canonical form NIP-IC gives; for a platform NIP-IC gives no canonical form, a client MUST NOT derive it from typed input and MUST use the key from the recipient's `35522` attestation or `nconnection` instead, and a request addressed to it names the IA that attested it in its `ia` tag. Native Nostr identities MUST NOT be hashed into a ConnectionKey — a real pubkey is used as-is.
- **Proxy Agent**: A bot or service that signs a `9523` event on behalf of a platform user whose keys it does not hold.
- **Chain**: The energy-backed coin, routable over the Lightning Network, that a zap settles on, named by the `chain` tag, e.g. `bitcoin`. Examples use `examplecoin` for any Lightning-routable chain other than Bitcoin.

## Specification

### Event Kinds

| Kind | Name | Signed by | Mechanism |
|------|------|-----------|-----------|
| `9520` | AltZap Request | Sender | LNURL-Pay |
| `9523` | On-Behalf AltZap Request | Proxy Agent | LNURL-Pay |
| `9521` | AltZap Receipt | LNURL server | — |

Kind `9522` is the Cash Zap, defined in [NIP-CZ](./NIP-CZ.md): the same zap paid with a cash token instead of an invoice. Kind `9525` is the Pocket Entry, defined in [NIP-CP](./NIP-CP.md).

### LNURL Chain Advertisement

An endpoint advertises the chains it accepts as `chain/<name>` entries in its LUD-06/LUD-16 `metadata` array; wallets that do not know them ignore them.

```json
{
  "callback": "https://example.com/lnurl/callback/<id>",
  "maxSendable": 100000000,
  "minSendable": 1000,
  "metadata": "[[\"text/plain\",\"...\"],[\"chain/examplecoin\",\"examplecoin\"]]",
  "tag": "payRequest",
  "allowsNostr": true,
  "nostrPubkey": "<lnurl-server-pubkey>"
}
```

- `allowsNostr: true` signals, as in NIP-57, that the callback accepts a signed Nostr event in its `nostr` query parameter. `nostrPubkey` is the pubkey that will sign the resulting receipt.
- Each `chain/<name>` entry in `metadata` advertises one supported chain; only the listed chains are supported. An endpoint with no `chain/<name>` entry settles on Bitcoin only and is a plain NIP-57 endpoint.
- An endpoint supports AltZap only if it has `allowsNostr: true` **and** at least one `chain/<name>` entry. A client MUST NOT send a `9520`/`9523` request to any other endpoint; it uses a NIP-57 `9734` request there instead.
- Amounts (`minSendable`/`maxSendable`, and the request's own `amount` tag) are denominated in that chain's milli-unit: `msats` for `chain/bitcoin`, and for any other chain (e.g. `chain/examplecoin`) the milli-unit that chain defines.

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

- `[1]` is a 64-character lowercase hex Nostr pubkey **or** a ConnectionKey. A receiver MUST reject any other value, uppercase hex included, and never lowercases it.
- `[2]` is the platform name (`discord`, `email`, `x`, ...). If omitted or empty, it MUST be interpreted as `nostr` — `[1]` is a native pubkey. A sender MUST write any other value in NIP-IC's platform-name format (one or more of `a`–`z`, `0`–`9`, `.`, `_` and `-`), and a receiver MUST reject a tag whose `[2]` does not, and never lowercases or otherwise repairs it.

`9521` receipts additionally allow a 4th element, `["p/P", "<hex-or-ConnectionKey>", "<platform>", "<handle>"]` — a stable, human-readable provider handle (e.g. a Discord username). It is informational only and MUST NOT be used for identity resolution.

### The `ia` Tag

A ConnectionKey on a platform outside NIP-IC's canonical table is defined by the Identity Authority (IA) that attested it, and only that IA's attestations resolve it to an account. A request addressed to such a key names that IA:

```
["ia", "<ia-pubkey>"]
```

- `[1]` is the IA's pubkey, 64-character lowercase hex.
- A sender MUST add exactly one `ia` tag to a `9520`/`9523` whose `p` is a ConnectionKey on a platform outside NIP-IC's canonical table, naming the IA whose `35522` attestation or `nconnection` it took the key from. It MAY add one when `p` is a ConnectionKey on a platform in that table. It MUST NOT add one when `p` is a Nostr pubkey.
- `ia` names the IA of `p` only. `P` addresses no value, and no tag names its IA.
- An LNURL server MUST reject a request whose `ia` names an IA it does not trust. It resolves a `p` that carries `ia` only through that IA's attestations, and a `p` without `ia` through any IA it trusts.
- The LNURL server copies the request's `ia` tag unchanged into the `9521` receipt.

### Kind 9520 — AltZap Request

```json
{
  "kind": 9520,
  "content": "Great post!",
  "tags": [
    ["e", "<optional-target-event-id>"],
    ["a", "<optional-kind:pubkey:d_tag>"],
    ["k", "<optional-target-event-kind>"],
    ["p", "<recipient-pubkey-or-ConnectionKey>", "<optional-platform>"],
    ["ia", "<ia-pubkey-if-p-is-a-ConnectionKey>"],
    ["amount", "1000000"],
    ["chain", "examplecoin"],
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

**Required tags**: `p` (exactly one), `ia` (exactly one when `p` is a ConnectionKey on a platform outside NIP-IC's canonical table), `amount` (positive integer, milli-units — `msats` for `bitcoin`, that chain's milli-unit otherwise), `chain`, `lnurl` (bech32 LNURL, see Verification Algorithm), `relays` (one or more `wss://`/`ws://` URLs).
**Optional tags**: `e` (at most one, 32-byte lowercase hex — the specific note being zapped), `a` (at most one, a `<kind>:<pubkey>:<d-tag>` coordinate, for zapping a parameterized replaceable event), `ia` (at most one, when `p` is a ConnectionKey on a platform in NIP-IC's canonical table), `k` (stringified kind number of the target event). A `9520` MUST NOT carry a `P` tag: the signer is the sender.

**Description-hash binding.** Once validated, the LNURL server issues a BOLT11 invoice whose `description_hash` equals the SHA256 of the event JSON exactly as received in the `nostr` parameter (after URL-decoding). The LNURL server MUST NOT re-serialize the event before hashing, and the receipt's `description` tag carries that same string verbatim.

### Kind 9523 — On-Behalf AltZap Request

Identical structure to `9520`, kind `9523` instead, with one additional constraint: **exactly one `P` (sender) tag is REQUIRED**. The event's own `pubkey` field (the Proxy Agent's signing key) MUST NOT be treated as the economic sender — implementations MUST read `P` instead.

```json
{
  "kind": 9523,
  "content": "Sent via Proxy Application",
  "tags": [
    ["p", "<recipient-ConnectionKey>", "<platform>"],
    ["P", "<sender-ConnectionKey>", "<platform>"],
    ["ia", "<optional-ia-pubkey>"],
    ["amount", "100000"],
    ["chain", "examplecoin"],
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
    ["ia", "<ia-pubkey-copied-from-the-request>"],
    ["amount", "<settled-milli-units>"],
    ["chain", "examplecoin"],
    ["bolt11", "<bolt11-invoice>"],
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
**Optional tags**: `e`/`a` (mirrored from the request), `ia` (copied from the request, present exactly when the request carries one), `amount` (the settled amount in milli-units, informational), `description` (the stringified original `9520`/`9523` event — omitted for a plain LNURL payment settled without any signed request), `r`/`R` (resolved native pubkeys, see below).

#### `r` / `R` — Resolved Identity

When the LNURL server can independently map a ConnectionKey back to a real Nostr pubkey (e.g. via a `kind:35521` identity connection it has indexed, verified through the IA in `ia` when the request carries one), it MAY attach that pubkey as `r` (for a ConnectionKey `p`) or `R` (for a ConnectionKey `P`).

#### Identity Tag Matrix

| Intent kind | Sender (`P`) | Recipient (`p`) | `r` | `R` |
|---|---|---|---|---|
| `9520` | Signer pubkey (no `P` on the request; the receipt sets it) | Pubkey or ConnectionKey | If `p` is a ConnectionKey and resolved | N/A — sender is already a pubkey |
| `9523` | ConnectionKey | ConnectionKey | If resolved | If resolved |

## Verification Algorithm

Two actors validate AltZap events, with different subsets:

- **LNURL server**: validates a `9520`/`9523` request at its callback before issuing an invoice. It MUST enforce every rule below.
- **Relay**: stores events someone else signed. A receipt records a payment that already happened, so a relay MUST reject an event that is forged or provably inconsistent, and SHOULD NOT reject one for a rule that only the LNURL server could have enforced. A relay MAY skip AltZap validation entirely.

Clients displaying a receipt SHOULD apply the relay subset. AltZap's rules are stricter than NIP-57's: implementations that share validator code between the two MUST NOT relax an AltZap MUST to a NIP-57 tolerance.

Signature verification MUST happen before any tag content is trusted; the remaining checks are independent and their order does not matter.

### Request (`9520` / `9523`)

| # | Rule | LNURL server | Relay |
|---|------|--------------|-------|
| 1 | `id`/`sig` verify against `pubkey` per NIP-01. | MUST | MUST |
| 2 | Exactly one `p` tag; the `[1]` of every `p` and `P` tag is 64-character lowercase hex, and its `[2]`, if neither empty nor `nostr`, meets NIP-IC's platform-name format. | MUST | MUST |
| 3 | `9520` has no `P` tag; `9523` has exactly one. | MUST | MUST |
| 4 | `chain` is present, non-empty, and one of the chains the endpoint advertises. | MUST | SHOULD (presence only) |
| 5 | `amount` is a positive integer and equals the callback's `amount` parameter. | MUST | SHOULD (positive integer only) |
| 6 | `lnurl` is present and well formed (see below), and decodes to this LNURL server's own endpoint. | MUST | MAY (well formed only) |
| 7 | `relays` has at least one `ws://`/`wss://` URL. | MUST | MAY |
| 8 | At most one `e` tag, a 32-byte ID in lowercase hex, and at most one `a` tag. | MUST | SHOULD |
| 9 | `ia` meets §The `ia` Tag: at most one, 64-character lowercase hex, present when `p` is a ConnectionKey on a platform outside NIP-IC's canonical table, absent when `p` is a Nostr pubkey. | MUST | SHOULD |
| 10 | `ia`, if present, names an IA the LNURL server trusts. | MUST | — |

**The `lnurl` tag** carries the recipient's LNURL-pay endpoint as a bech32 string with the `lnurl` prefix (LUD-01), not a lightning address. Bech32 is case-insensitive: receivers MUST accept `lnurl1…` and `LNURL1…` alike and compare decoded URLs, not strings. It is required on `9520` and `9523` (optional in NIP-57): an LNURL server MUST reject a request whose decoded `lnurl` is not its own endpoint.

### Receipt (`9521`)

| # | Rule | Publisher (LNURL server) | Relay |
|---|------|--------------------------|-------|
| 1 | `id`/`sig` verify against `pubkey` per NIP-01. | MUST | MUST |
| 2 | `bolt11`, `chain` and `preimage` are present. | MUST | MUST |
| 3 | Hash lock: `SHA256(preimage)` equals the invoice's payment hash. | MUST | MUST |
| 4 | The invoice belongs to the network named in `chain`. | MUST | MUST |
| 5 | If `description` is present: it parses as JSON, embeds a validly signed `9520`/`9523` request, and `SHA256(description)` equals the invoice's `description_hash`. | MUST | MUST |
| 6 | If `description` is present: the receipt's `p`, `chain` and `ia` equal the embedded request's (an `ia` absent from both is equal), and its `P` equals the embedded `9523`'s `P`; for an embedded `9520`, which has no `P`, a receipt `P` names that request's `pubkey`. `p` and `P` are compared by `[1]` and `[2]`, ignoring the receipt's handle. | MUST | MUST |
| 7 | If `description` is present: the invoice amount equals the embedded request's `amount`. | MUST | SHOULD |
| 8 | If `description` is absent (a plain LNURL payment settled without a signed request): the invoice amount is positive. | MUST | SHOULD |
| 9 | If `amount` is present, it equals the invoice amount. | MUST | SHOULD |
| 10 | If `description` is present: the receipt's `pubkey` equals the `nostrPubkey` advertised by the endpoint the embedded request's `lnurl` decodes to. | MUST | MAY |

A client MUST apply receipt rule 10 before it displays a receipt as a payment: anyone who learns a paid invoice's preimage can sign a receipt that passes rules 1 to 9 with their own key.

## Custodial Settlement

An LNURL server MAY act as a temporary custodian when the resolved recipient has no reachable wallet at settlement time — either because `p` is a ConnectionKey with no linked identity yet, or because a linked Nostr wallet is unresponsive. In both cases the LNURL server publishes the `9521` receipt as normal and holds the settled value internally against the recipient's ConnectionKey or pubkey until they authenticate and connect a wallet, at which point the LNURL server sweeps the balance to them. For a `p` that carries `ia`, the LNURL server MUST hold the value against that ConnectionKey and IA together, and sweep it only to the pubkey that IA's current attestation for the key names; a `p` without `ia` is swept to the pubkey a trusted IA's attestation names. Custody is otherwise an operational behavior of the LNURL server, with no event of its own.

## Backwards Compatibility

AltZap is **not** wire-compatible with NIP-57: a client that only understands `9734`/`9735` neither generates nor recognizes `9520`/`9521`/`9523` events, and vice versa. An LNURL server serving both runs both flows side by side, keyed on kind number.

## Security Considerations

- **Cross-chain replay**: a request is settled only on the chain its `chain` tag names; an LNURL server rejects a mismatch and never substitutes another chain.
- **Proxy trust boundary**: `9523`'s signing `pubkey` is the Proxy Agent, not the sender. Clients and LNURL servers MUST read `P`, not `pubkey`, when displaying or attributing the sender.
- **ConnectionKey collisions**: a ConnectionKey is a one-way hash of `"<platform>:<platform-user-id>"`. Implementations MUST NOT accept a claimed reverse mapping without independent verification (e.g. a signed identity-connection event) — the receipt alone is not proof of ownership.
- **`r`/`R` disclosure**: resolving a ConnectionKey to a real pubkey in a receipt is a deliberate, visible linkage of a Web Identity to a Nostr identity. An LNURL server SHOULD only populate `r`/`R` when it has verified authorization to make that link public.
