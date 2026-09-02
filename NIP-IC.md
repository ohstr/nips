NIP-IC
======

Identity Connection: Legacy Account Linking
---------------------------------------------

`draft` `optional`

**Depends on**: [NIP-01](https://github.com/nostr-protocol/nips/blob/master/01.md), [NIP-40](https://github.com/nostr-protocol/nips/blob/master/40.md) (Expiration Timestamp), [NIP-09](https://github.com/nostr-protocol/nips/blob/master/09.md) (Event Deletion)
**Related**: [NIP-AZ](./NIP-AZ.md) (AltZap) — consumes this NIP's Kind 35521 to populate the `r`/`R` resolved-identity tags on an AltZap Receipt.

## Abstract

This NIP defines two event kinds — Kind `35521` (Identity Connection) and Kind `35522` (IA Attestation) — that let a user cryptographically bind their Nostr public key to an account on a non-Nostr platform (a **Legacy Identity Provider**, LIDP: Discord, X, email, ...), witnessed by a permissionless, federated network of **Identity Authorities** (IAs). A deterministic verification-state model lets any client resolve a ConnectionKey — the hash NIP-AZ uses to address a recipient who has no Nostr keypair yet — back into a verified Nostr identity, without trusting the claim on its own.

## Motivation

A self-signed claim of "this Nostr key is mine, and I also own this Discord account" is worthless on its own — anyone can sign that claim for any account they don't own. Something has to have actually checked the claim before publishing it.

This NIP splits the claim into two signed layers instead of one:

1. **The user's own claim** (Kind 35521) — cheap, replaceable, and can be freely re-published, but by itself proves nothing.
2. **A third party's witnessed proof** (Kind 35522) — a signature from an Identity Authority that actually performed the verification (checked a public post, sent an OTP, etc.) and is willing to stake its own reputation on the result.

Splitting these apart — rather than having a single self-attested event — means the *verification logic* (challenge issuance, evidence checking) is decoupled from the *identity graph* (which pubkey claims which account). Multiple independent IAs can attest the same connection without coordinating with each other, and a client only needs to trust **one** of them.

## Terminology

- **IA (Identity Authority)**: A service that verifies a user's ownership of an LIDP account and signs a Kind `35522` attestation. Anyone can run one — there is no central gatekeeper.
- **LIDP (Legacy Identity Provider)**: A non-Nostr identity platform (Discord, Telegram, X, GitHub, Instagram, Facebook, a DNS domain, or email). See [NIP-AZ](./NIP-AZ.md) for the shared LIDP/ConnectionKey terminology.
- **ConnectionKey**: `SHA256("<lidp>:<lidp-user-id>")` — see NIP-AZ. The `d` tag of both event kinds in this NIP.
- **Deep Check**: Fetching the Kind `35522` attestation from its origin relay (rather than trusting an unverified reference) to confirm it is genuinely signed, unexpired, and unrevoked.
- **Challenge Token (`npv1`)**: A bech32-encoded, session-bound proof string a user publishes publicly on the LIDP platform as evidence of control.

## Specification

### Event Kinds

| Kind | Name | Signed by | Replaceable |
|------|------|-----------|-------------|
| `35521` | Identity Connection | The end user | Yes (parameterized, per `d` tag) |
| `35522` | IA Attestation | The Identity Authority | Yes (parameterized, per `d` tag) |

Both are kind-30000-range parameterized replaceable events per NIP-01: a new event with the same `pubkey` + `d` tag replaces the prior one.

### Kind 35521 — Identity Connection

```json
{
  "kind": 35521,
  "pubkey": "<user-pubkey>",
  "created_at": "<unix-timestamp>",
  "tags": [
    ["d", "<connection-key>"],
    ["e", "<attestation-event-id>", "<ia-relay-url>"],
    ["lidp", "<lidp-name>"]
  ],
  "content": "<public-profile-json>",
  "id": "...",
  "sig": "..."
}
```

Published by the end user to claim a connection. On its own this is an unverified claim — see Verification Model.

**Required tags**: `d` (the bare ConnectionKey — **MUST NOT** carry a platform prefix like `discord:`; a prefixed `d` tag breaks `#d` filter queries from third-party clients), `lidp` (the LIDP name, for relay-side filtering), `e` (references the witnessing `35522`; see below).

**`e` tag**: `["e", "<attestation-event-id>", "<ia-relay-url>"]` — a **reference**, not an embed. The attestation is fetched from `<ia-relay-url>` on demand, keeping the `35521` event small (~100 bytes vs. 1-2 KB for an inlined attestation). A user MAY include multiple `e` tags, one per IA that has independently attested the same connection ("stacking") — a client only needs to trust one of them.

**`content`**: cleartext JSON profile metadata (`display_name`, `picture`, `user_id`, `username`) — a display convenience copied from the IA's evidence, **not authoritative**. The authoritative `user_id`/`username` live in the referenced `35522`'s `evidence` tag; a client that renders `content` without cross-checking it against the fetched attestation is trusting an unverified, user-controlled field (see Security Considerations).

### Kind 35522 — IA Attestation

```json
{
  "kind": 35522,
  "content": "",
  "tags": [
    ["d", "<connection-key>"],
    ["p", "<user-pubkey>"],
    ["lidp", "<lidp-name>"],
    ["evidence", "<verification-payload-json>"],
    ["expiration", "<unix-timestamp>"]
  ],
  "pubkey": "<ia-pubkey>",
  "created_at": "<unix-timestamp>",
  "id": "...",
  "sig": "..."
}
```

Signed by the IA immediately upon completing a successful verification flow. `content` is always empty — everything relevant is in the tags.

**Required tags**: `d` (same ConnectionKey as the `35521` it will be referenced by), `p` (the Nostr pubkey that proved ownership — a client MUST verify this equals the outer `35521`'s `pubkey` when cross-checking, to prevent an attestation issued to one user being replayed inside a different user's `35521`), `lidp`, `evidence` (see below). `expiration` is a [NIP-40](https://github.com/nostr-protocol/nips/blob/master/40.md) timestamp; a default of 90 days is RECOMMENDED, and an IA MAY omit it for a non-expiring attestation.

#### The `evidence` Tag

A cleartext JSON (v1) string:

| Field | Type | Description |
|---|---|---|
| `version` | number | Always `1`. |
| `lidp` | string | LIDP name. |
| `auth_type` | string | Always `"public_post"` — every supported verification mode requires a public post of the challenge token. |
| `user_id` | string | The normalized LIDP account identifier. |
| `username` | string | The provider handle at verification time. |
| `verified_at` | number | Unix timestamp of successful verification. |
| `evidence_url` | string | URL of the public post carrying the challenge. |
| `challenge` | string | The `npv1...` challenge token published at `evidence_url`. |
| `pre_auth_code` | string | The raw code the challenge was derived from — REQUIRED for cross-IA re-verification (see below). |

Because verification is backed by a public, independently-fetchable URL, any IA can re-verify the same claim without user interaction — simply fetch `evidence_url` and confirm `challenge` is present.

#### The `npv1` Challenge Token

A bech32-encoded TLV binding a challenge to one specific user and session, preventing replay across sessions:

```
type=0x00 || length=0x20 || value=SHA256(pubkey_bytes || pre_auth_code_bytes)
```

A second IA performing cross-IA re-verification from existing evidence MUST:
1. Bech32-decode `challenge` to its 34-byte TLV payload.
2. Strip the 2-byte header (`0x00 0x20`) to recover the 32-byte session hash.
3. Recompute `SHA256(hex.Decode(p_tag_pubkey) || evidence.pre_auth_code)`.
4. Assert the two hashes are equal.

Step 4 is what distinguishes a genuinely session-bound challenge from a token replayed out of a different session — which is why `pre_auth_code` MUST always be present in `evidence`; without it, step 3 cannot be performed.

### The `nconnection` Bech32 Encoding

A portable, shareable string (e.g. handed from a Discord bot to a mobile wallet) bundling a ConnectionKey with the relays where its `35521` events are published, structured as TLV — the same pattern [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) uses for `nprofile`/`nevent`:

| Type | Name | Length | Description |
|---|---|---|---|
| `0` | Special | 32 bytes | The raw ConnectionKey bytes (hex-decoded). |
| `1` | Relay | variable | A relay URL, UTF-8. Repeatable. |

`nconnection` is **not** a registered NIP-19 prefix — it is a distinct, protocol-specific encoding that happens to reuse NIP-19's TLV convention for consistency.

## Verification Model

A client holding a 3-element identity tag (e.g. `["p", "<ConnectionKey>", "discord"]`, per NIP-AZ) resolves it to a Nostr identity as follows:

1. Query `{"kinds": [35521], "#d": ["<ConnectionKey>"]}` against the network (and specifically the ZSP's own relays, if known).
2. If no event is returned, the connection is **unverified** — fall back to displaying the LIDP-native profile (e.g. avatar/username from the ZSP's own API).
3. If a `35521` is returned, extract its `e` tag and fetch the referenced `35522` from the named relay hint (a Deep Check).
4. If the fetch succeeds, the event is validly signed by its claimed `pubkey`, unexpired, and unrevoked (see Event Lifecycle), the connection is **Verified**. If the `35521` exists but its `35522` is missing, invalid, expired, or revoked, the connection is **Spoofed** — this is distinct from *unverified*: someone published a `35521` claim that does not check out.

| State | Condition | Recommended UI |
|---|---|---|
| **Verified** | `35521` found; its `35522` fetches and checks out | ✅ native Nostr profile |
| **Pending (fallback)** | No `35521` found | ⏳ fallback LIDP profile |
| **Spoofed** | `35521` found; `35522` missing, invalid, expired, or revoked | ⚠️ flag as unverified/suspicious |

## Event Lifecycle

1. **Creation**: the user completes a verification flow with an IA; the IA signs `35522`.
2. **Publishing**: the IA publishes `35522` to its own relay(s). Session state: `confirmed`. **Routing is NOT yet active.**
3. **Linking**: the IA hands the raw `35522` event back to the user's client, which extracts the event ID + IA relay URL, builds the `e` tag, and publishes `35521`. The attestation itself is never embedded.
4. **Activation**: after publishing `35521`, the client signals the IA (mechanism is implementation-defined). Only now does the IA record the LIDP→Nostr routing link and move the session to `active`. If the user abandons the flow before this step, no routing link is ever created.
5. **Revocation**: to revoke, the IA publishes a [Kind 5](https://github.com/nostr-protocol/nips/blob/master/09.md) deletion targeting the `35522` ID. This immediately fails any subsequent Deep Check. If the session was `active`, the IA also removes its internal routing record.

## Federated Trust Model & Multi-IA Stacking

Zapf's IA layer is federated, not centralized: anyone can run an IA, and different wallets/clients choose which IAs they trust — there is no protocol-level allowlist. A user can strengthen their identity by having *multiple* independent IAs attest the same LIDP account, referencing all of them via multiple `e` tags on one `35521`. A client only needs to trust one attesting IA to consider the connection verified, which gives resilience against any single IA going offline or being compromised.

## Security Considerations

- **Content-field spoofing**: `35521`'s `content` is user-controlled and unverified. A ZSP MUST route payments strictly by the cryptographic `d`-tag hash, never by `content` — a user cannot redirect funds by editing their display name. A verifying client MUST cross-check `content` claims against the fetched `35522`'s `evidence` and flag a mismatch as spoofed, rather than rendering `content` at face value.
- **Self-attestation**: a `35522` is only meaningful because it's signed by a key distinct from the user's own — a verifier MUST confirm the attesting `pubkey` is a key it (or its trust set) recognizes as an IA, not the user's own key.
- **Cross-session replay**: the `npv1` challenge binds a specific `pubkey` + `pre_auth_code` pair (see above); without checking that binding, a challenge published for one verification session could be replayed to justify an unrelated one.
- **`p`-tag confusion attack**: an attacker could copy a real `35522` (issued to user A) and reference it from a `35521` published by user B. A verifying client MUST check that the attestation's `p` tag equals the outer `35521` event's own `pubkey` — a mismatch means the attestation does not belong to this connection, regardless of how it was referenced.
- **No private mode**: `evidence.user_id`/`evidence.username` are always cleartext and public once a `35522` is published. There is no way to link an identity without disclosing these fields on the relay.
- **Attestation expiry**: an IA SHOULD default to a bounded `expiration` (90 days is RECOMMENDED) rather than an unexpiring attestation, so a compromised or abandoned IA's attestations naturally age out even if the IA never explicitly revokes them.

## Relation to Other NIPs

**NIP-AZ**: an AltZap Receipt's `r`/`R` tags (resolved recipient/sender pubkey) are populated by exactly the lookup this NIP defines — a ZSP that has indexed `35521` connections can resolve a ConnectionKey `p`/`P` tag to a real pubkey and attach it to the receipt.

**NIP-40**: the `35522` `expiration` tag reuses NIP-40's semantics unmodified — relays that already prune on NIP-40 expiration need no additional logic to age out attestations.

**NIP-09**: attestation revocation is a standard Kind 5 deletion targeting the `35522` event ID; no new deletion mechanism is introduced.

**NIP-19**: the `nconnection` encoding reuses NIP-19's TLV pattern for consistency but is not a NIP-19-registered entity type.

## Reference Implementation

Go: [`github.com/ohstr/zapf`](https://github.com/ohstr/zapf) — `internal/domain`, `internal/nostr`, `internal/service` (`cross_attest.go`, `verification.go`), `pkg/nostr/attestation_event.go`. Not yet extracted into a standalone, reusable `nmilat` package the way NIP-AZ is.

## Documentation

Deployed protocol docs and guides: [docs.zapf.app](https://docs.zapf.app)

- [Identity Authority](https://docs.zapf.app/concepts/identity-authority)
- [Legacy Identity Providers](https://docs.zapf.app/concepts/legacy-identity-providers)
- [Verification Model](https://docs.zapf.app/concepts/verification-model)
- [Kind 35521 spec](https://docs.zapf.app/protocol/identity-connection-35521)
- [Kind 35522 spec](https://docs.zapf.app/protocol/ia-attestation-35522)
- [Evidence Sharing guide](https://docs.zapf.app/guides/evidence-sharing)
- [Running an IA guide](https://docs.zapf.app/guides/running-an-ia)
