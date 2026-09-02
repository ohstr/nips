# Identity Connection: Kind 35521 & Kind 35522

Complete reference for the Web-Identity-to-Nostr identity binding protocol. Load this file when building or auditing anything involving Kind 35521, Kind 35522, evidence format, ConnectionKey derivation, or `nconnection` URL encoding.

For the full session flow (VerifyFlow, verification service, state machines), see `../connection-web-identity-flow/SKILL.md`.

---

## A — ConnectionKey Derivation

```
ConnectionKey = lowercase_hex(SHA256(platform + ":" + platformUserID))
```

| Input | Output |
|-------|--------|
| `"discord"` + `"123456789"` | 64-char lowercase hex |
| `"github"` + `"user42"` | different 64-char hex |

Go implementation:

```go
// pkg/nostr/connection_event.go
func GenerateConnectionKey(platform, externalID string) string {
    h := sha256.Sum256([]byte(platform + ":" + externalID))
    return hex.EncodeToString(h[:])
}
```

**Rules:**
- NEVER include the platform name as a prefix in the `d` tag of Kind 35521 or 35522. The `d` tag must be the bare ConnectionKey: `["d", connectionKey]`, never `["d", "discord:connectionKey"]`.
- Nostr pubkeys are NEVER hashed into ConnectionKeys. Native Nostr identifiers (pubkeys, NIP-05 addresses) are used directly, not hashed.
- The ConnectionKey is fully deterministic: same `platform` + `platformUserID` always produces the same key.
- The `platform` tag in both kinds carries the provider name separately from the `d` tag.

---

## B — Evidence Format (Kind 35522 `evidence` tag)

The `evidence` tag value is a **cleartext JSON string** (v1 format). It is always readable by anyone — there is no encryption. This enables cross-IA trustlessness: any IA can re-verify the claim by fetching `evidence_url` and confirming the challenge token.

### Fields (v1)

| Field | Type | Description |
|-------|------|-------------|
| `version` | `number` | Always `1` |
| `platform` | `string` | Web Identity platform name (e.g., `"discord"`) |
| `auth_type` | `string` | Always `"public_post"` — all flows require a public challenge post |
| `user_id` | `string` | Normalized platform account identifier |
| `username` | `string` | Provider handle at verification time |
| `verified_at` | `number` | Unix timestamp of verification |
| `evidence_url` | `string` | URL of the public post where the challenge appeared |
| `challenge` | `string` | The `npv1…` challenge token published at `evidence_url` |
| `pre_auth_code` | `string` | Raw pre-auth code for cross-IA challenge binding |

### Complete v1 example

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

Frontend parsing: `parseEvidenceTag(raw)` in `NostrConnectionRepository.ts` — calls `JSON.parse(raw)` and validates `version === 1`.

---

## C — Kind 35521 Content Field

The `content` field of Kind 35521 is always **cleartext JSON** containing profile display data sourced from the IA attestation. It does NOT store the ConnectionKey or attestation — the `e` tag carries the attestation reference.

```json
{ "display_name": "alice", "picture": "https://...", "user_id": "123456789", "username": "alice" }
```

| Field | Source | Description |
|-------|--------|-------------|
| `display_name` | IA profile data | Human-readable name on the platform |
| `picture` | IA profile data | Avatar URL |
| `user_id` | Kind 35522 `evidence` tag | Platform-specific user ID |
| `username` | Kind 35522 `evidence` tag | Platform handle / username |

`user_id` and `username` are a **convenience copy** for quick display — authoritative values live in the Kind 35522 `evidence` tag. The content may be empty (`""`) if no profile data was provided.

Note: `encryptedId` (`data.encryptedId`) is a separate optional NIP-44 self-encrypted field for the raw platform user ID — it is distinct from the content field and only used when the user explicitly requests it.

---

## D — `nconnection` Bech32 Encoding

Used to generate profile page URLs for unresolved Web Identities that have not yet published a Kind 35521.

**Format:** `nconnection1{bech32-encoded TLV data}`

**TLV structure:**

| Type | Size | Content |
|------|------|---------|
| 0 | 32 bytes | ConnectionKey raw bytes (hex-decoded) — **required** |
| 1 | variable | Relay URL as UTF-8 bytes — repeatable, one per relay hint |
| 2 | variable | Platform name as UTF-8 bytes (e.g., `"discord"`) — at most once |

Implementation: `frontend/src/utils/nconnection.ts`

```typescript
export function encodeNconnection(
    connectionKey: string,   // 64-char hex
    relays: string[] = [],
    platform?: string
): string

export function decodeNconnection(str: string): {
    connectionKey: string;
    relays: string[];
    platform?: string;
} | null
```

Used in `useProfileLink` (`frontend/src/hooks/useProfileLink.ts`) when `isResolved === false`:
```typescript
return '/' + encodeNconnection(identifier, [], platform);
```

Profile pages at `/nconnection1...` display the platform icon and username if available, and prompt the user to verify to claim their identity.

---

## E — Cross-IA Challenge Binding

Cross-IA re-attestation does NOT use a Schnorr `ia_signature`. It uses the `pre_auth_code` field in the cleartext evidence JSON together with the `npv1` challenge token.

**Algorithm** (any IA or client):
1. Bech32-decode the `challenge` token to get the 34-byte TLV payload.
2. Strip the 2-byte TLV header (`0x00 0x20`) → 32-byte session hash.
3. Compute `SHA256(hex.Decode(p_tag_pubkey) || []byte(evidence.pre_auth_code))`.
4. Assert both hashes match.

This proves the challenge was genuinely bound to this user and session — not replayed from another. Without `pre_auth_code`, step 3 is impossible, which is why it must always be included in the evidence payload.

Implementation: `internal/service/cross_attest.go` / `pkg/nostr/attestation_event.go` — `EvidencePayload.PreAuthCode`.

---

## F — Security Rules

1. **p-tag cross-check** — Always verify `attestation.p_tag === outerKind35521.pubkey` before trusting a Kind 35522 referenced by a Kind 35521. Prevents embedding attacks: an attacker could copy a valid Kind 35522 event ID (issued to user A) and put it in the `e` tag of user B's Kind 35521.

   ```typescript
   // frontend/src/infrastructure/repositories/NostrConnectionRepository.ts
   if (att.issuedToPubkey && att.issuedToPubkey !== ev.pubkey) continue;
   ```

2. **verifyEvent before trust** — Always call `verifyEvent(attestationEvent)` before parsing any Kind 35522. A forged signature silently invalidates the attestation.

3. **Expiration check** — If the `expiration` tag is present and its value `< Math.floor(Date.now() / 1000)`, treat the attestation as `state: 'expired'`, not `'verified'`. Default expiry: 90 days from issuance (configurable via `IA_ATTESTATION_EXPIRY_DAYS`).

4. **Revocation check (deep verify)** — Re-fetch the Kind 35522 from the IA relay (URL from the `e` tag) using `ONLY_RELAY` cache mode (bypasses IndexedDB). If the relay no longer returns the event, treat as `state: 'revoked'`. Implemented in `deepVerifyAttestation` in `NostrConnectionRepository.ts`. Note: the initial `fetchConnections` relay fetch already sets state to `'verified'` — `deepVerifyAttestation` is an explicit re-check for liveness.

5. **d-tag format** — Any Kind 35522 whose `d` tag contains a `:` separator (e.g., `"discord:abc123"`) is invalid. `ValidateAttestation` in `internal/nostr/validator.go` enforces this.

---

## G — Full Event Reference

### Kind 35522 — IA Attestation

```json
{
  "kind": 35522,
  "pubkey": "<ia_pubkey>",
  "content": "",
  "tags": [
    ["d", "<connection_key>"],
    ["p", "<user_nostr_pubkey>"],
    ["platform", "<provider>"],
    ["evidence", "<v1_cleartext_json>"],
    ["expiration", "<unix_timestamp>"]
  ]
}
```

Signed by IA private key via `CreateAttestationEvent(params)` in `pkg/nostr/attestation_event.go`.

### Kind 35521 — User Identity Connection

```json
{
  "kind": 35521,
  "pubkey": "<user_nostr_pubkey>",
  "content": "{\"display_name\":\"alice\",\"user_id\":\"123456789\",\"username\":\"alice\"}",
  "tags": [
    ["d", "<connection_key>"],
    ["platform", "<provider>"],
    ["e", "<attestation_event_id>", "<ia_relay_url>"],
    ["expiration", "<unix_timestamp>"]
  ]
}
```

- `e` tag: 3-element reference to Kind 35522 — consumers must fetch it from `ia_relay_url` to verify. **Never embed the full JSON.**
- `expiration`: optional, mirrored from the Kind 35522 `expiration` tag (NIP-40) so relays can auto-prune expired events.
- `content`: always cleartext JSON. `user_id` and `username` are copied from the evidence for quick display without a relay fetch.
- Multiple `e` tags allowed — one per IA that attested the same connection (multi-IA stacking).

### Kind 5 — Revocation (NIP-09)

```json
{
  "kind": 5,
  "pubkey": "<ia_pubkey>",
  "content": "revoked",
  "tags": [["e", "<attestation_event_id>"]]
}
```

Published by the IA via `RevokeAttestation(sessionID)` in `internal/service/verification.go`. Signed by the same IA key that issued the Kind 35522.
