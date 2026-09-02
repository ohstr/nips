# NIP-AZ (AltZap): Event Kinds & Data Formats

## The 3-Element Tag Standard
To ensure universal legacy identity resolution without a hard requirement for a Nostr wallet, all identity tags (`p` for Recipient, `P` for Sender) MUST use the following array structure:
`["p/P", "<hex_or_ConnectionKey>", "<platform_name>"]`

- **[0] Key**: `p` (Recipient), `P` (Sender)
- **[1] Identity**: 64-character hex pubkey OR ConnectionKey.
- **[2] Provider**: The source. Used values: `nostr`, `discord`, `email`, `x`.

## Kind 5520: Native AltZap Request

```json
{
  "kind": 5520,
  "content": "Here is 1000 loki! Great post.",
  "tags": [
    ["e", "<optional_target_event_id>"],
    ["a", "<optional_kind:pubkey:d_tag>"],
    ["k", "<optional_target_event_kind>"],
    ["p", "<recipient_pubkey_or_ConnectionKey>", "<optional_platform_name>"],
    ["amount", "1000000"],
    ["chain", "flokicoin"],
    ["lnurl", "lnurl1..."],
    ["relays", "wss://relay.ohstr.com", "wss://relay.primal.net"]
  ],
  "pubkey": "<sender_pubkey>",
  "created_at": "<unix_timestamp>",
  "id": "...",
  "sig": "..."
}
```

Used by native Nostr users to zap another user or LNURL destination.
* **Initiator:** Signed by Sender's Private Key.
* **Mechanism:** LNURL-Pay.
* **Required Tags:**
    * `p`: Recipient identity (e.g., `["p", "<recipient_pubkey_or_ConnectionKey>", "<optional_platform_name>"]`). If the 3rd parameter is omitted or empty, it defaults to `nostr` (pubkey).
    * `amount`: Total value in milli-units.
    * `lnurl`: Bech32 LNURL of the recipient.
    * `relays`: Target relays.
    * `chain`: `flokicoin` or `bitcoin`.
* **Optional Tags:**
    * `e`: Target event ID if zapping a specific note.
    * `a`: Target coordinate if zapping a replaceable event.
    * `k`: Stringified kind of the target event.

## Kind 5523: On-Behalf AltZap Request (Proxy)

```json
{
  "kind": 5523,
  "content": "Sent via Proxy Application",
  "tags": [
    ["p", "<recipient_ConnectionKey>", "<platform_name>"],
    ["P", "<sender_ConnectionKey>", "<platform_name>"],
    ["amount", "100000"],
    ["chain", "flokicoin"],
    ["relays", "wss://relay.zapf.app", "wss://relay.damus.io"],
    ["lnurl", "lnurl..."]
  ],
  "pubkey": "<bot_server_identity>",
  "created_at": 1709424000,
  "sig": "..."
}
```

Used by bots (Proxy Agents) to represent users on legacy platforms.
* **Initiator:** Signed by Proxy Agent's Key (e.g., Zapf Discord Bot).
* **Mechanism:** LNURL-Pay.
* **Required Tags:**
    * `P`: True Sender identity (3-element format, e.g., `["P", "<sender_ConnectionKey>", "<platform_name>"]`).
    * `p`: Recipient identity (3-element format, e.g., `["p", "<recipient_ConnectionKey>", "<platform_name>"]`).
    * `amount`: Value in milli-units.
    * `lnurl`: Recipient LNURL endpoint.
    * `chain`: Network context.
    * `relays`: Target relays for receipt publishing.

**Validation / Identity Resolution (via Kind 35521):**
When parsing a `5523` event (or its resultant `5521`), the ConnectionKey is the primary identifier. To display a "Verified" UI experience, clients MUST query for a **Kind 35521 (Identity Connection)** where the `#d` tag matches the ConnectionKey. If no 35521 exists, display the fallback Web Identity (e.g., Provider Avatar).

## Kind 5521: Universal AltZap Receipt

```json
{
  "kind": 5521,
  "content": "",
  "tags": [
    ["e", "<optional_target_event_id>"],
    ["a", "<optional_kind:pubkey:d_tag>"],
    ["p", "<recipient_hex_or_ConnectionKey>", "<platform_name>", "<handle>"],
    ["P", "<sender_hex_or_ConnectionKey>", "<platform_name>", "<handle>"],
    ["r", "<resolved_recipient_nostr_pubkey>"],
    ["R", "<resolved_sender_nostr_pubkey>"],
    ["amount", "<payment_amount_milli_units>"],
    ["chain", "flokicoin"],
    ["bolt11", "lnfc10u1..."],
    ["description", "<json_string_of_zap_request_optional>"],
    ["preimage", "<32_byte_hex_string>"]
  ],
  "pubkey": "<provider_pubkey>",
  "created_at": "<unix_timestamp>",
  "id": "...",
  "sig": "..."
}
```

The final proof of settlement issued by the ZSP.
* **Initiator:** Signed by ZSP's Private Key.
* **Required Tags:**
    * `p`: Mirror Recipient. The optional 4th element is the stable provider handle.
    * `P`: Mirror Sender. The optional 4th element is the stable provider handle. This structure supports proxy/anonymous routing.
    * `amount`: Settled milli-units extracted from the BOLT11.
    * `chain`: Settled network.
    * `bolt11`: The exact Lightning invoice that was paid.
    * `description`: Stringified JSON of the Request event (5520/5523).
    * `preimage`: Cryptographic proof (32-byte hex).
* **Optional Tags:**
    * `e`: Used if the AltZap is in response to a specific Nostr event (copied from request).
    * `a`: Used if the AltZap is in response to a parameterized replaceable event (copied from request).
    * `r`: Resolved Recipient (Used to map a `p` ConnectionKey back to a native pubkey if the ZSP knows the 35521 connection).
    * `R`: Resolved Sender (Used to map a `P` ConnectionKey back to a native pubkey if the ZSP knows the 35521 connection). Enables "Social Bridge" UX.

### Identity Tag Matrix for Agents (5521 Generation)

When generating a 5521 AltZap Receipt, use this matrix to correctly populate identity tags based on the parent intent kind:

| Intent Kind | Sender (`P`) | Recipient (`p`) | Resolved Recipient (`r`) | Resolved Sender (`R`) |
| --- | --- | --- | --- | --- |
| **5520** | **Pubkey** | Pubkey OR ConnKey | If `p` is ConnKey | N/A (Already Pubkey) |
| **5523** | **ConnKey** | **ConnKey** | If resolved | **If resolved** |

---

## Kind 35521: Identity Connection (User-Signed)

```json
{
  "kind": 35521,
  "pubkey": "<user_pubkey>",
  "created_at": "<unix_timestamp>",
  "tags": [
    ["d", "<connection_key>"],
    ["e", "<attestation_event_id>", "<ia_relay_url>"],
    ["platform", "<platform_name>"]
  ],
  "content": "<public_profile_json>",
  "id": "...",
  "sig": "..."
}
```

The user's public declaration that they own a Web Identity account. A parameterized replaceable event (30000–39999): keyed by `pubkey + d-tag`.

- `d` tag: bare ConnectionKey — **NEVER** prefixed with `platform:`. Including a prefix breaks `#d` relay queries from third-party clients.
- `platform` tag: provider name (e.g., `"discord"`, `"github"`)
- `e` tag: `["e", "<attestation_event_id>", "<ia_relay_url>"]` — 3-element reference to Kind 35522. Consumers must fetch the full event from the IA relay to verify. **Never embed the attestation JSON.**
- `expiration` tag: optional — mirrored from the Kind 35522 expiration (NIP-40) so relays can auto-prune expired events.
- `content`: always cleartext JSON — `display_name`, `picture`, `user_id`, `username`. Empty (`""`) if no profile data.
- Multiple `e` tags allowed — one per IA attesting the same connection (multi-IA stacking)

When querying relays: use `#d` filter with the bare ConnectionKey. Use `#platform` to filter by provider.

## Kind 35522: IA Attestation (IA-Signed)

```json
{
  "kind": 35522,
  "content": "",
  "tags": [
    ["d", "<connection_key>"],
    ["p", "<user_pubkey>"],
    ["platform", "<platform_name>"],
    ["evidence", "<verification_payload>"],
    ["expiration", "<unix_timestamp>"]
  ],
  "pubkey": "<ia_pubkey>",
  "created_at": "<unix_timestamp>",
  "id": "...",
  "sig": "..."
}
```

The Identity Authority's cryptographic proof that a user owns a Web Identity account.

- `d` tag: bare ConnectionKey (same as Kind 35521 `d` tag)
- `p` tag: user's Nostr pubkey — **frontend MUST verify `p == outerKind35521.pubkey`** to prevent embedding attacks
- `platform` tag: provider name
- `evidence` tag: `["evidence", "<verification_payload>"]` — cleartext v1 JSON string. Readable by anyone; enables cross-IA re-verification.
- `expiration` tag: NIP-40 unix timestamp — default 90 days, configurable via `IA_ATTESTATION_EXPIRY_DAYS`
- Content: `""` (empty)
- Signed by IA private key. Always call `verifyEvent(event)` before trusting.

### Evidence JSON Schema
A complete v1 evidence payload looks like this:
```json
{
  "version": 1,
  "platform": "discord",
  "auth_type": "public_post",
  "user_id": "1254093577051574374",
  "username": "joyosar",
  "verified_at": 1779219590,
  "evidence_url": "https://discord.com/channels/.../1506380827804831834",
  "challenge": "npv11qqsykd7ufyvfjl9qasdgtrz02jsv97l9atdnqc0vz8wsytxqxn9v6pqzvtph4",
  "pre_auth_code": "feb7dee63337"
}
```

## Kind 5: Attestation Revocation (NIP-09)

Published by the IA to revoke a Kind 35522. Clients performing deep verification must check for this.

- Tags: `["e", attestationEventId]`
- Content: `"revoked"`
- Signed by the same IA key that issued the Kind 35522

For full event JSON, encryption details, and security rules, see `identity-connection.md`.
