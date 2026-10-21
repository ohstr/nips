NIP-IC
======

Identity Connection: Web Identity Linking
-----------------------------------------

`draft` `optional`

**Depends on**: [NIP-09](https://github.com/nostr-protocol/nips/blob/master/09.md) (Event Deletion), [NIP-40](https://github.com/nostr-protocol/nips/blob/master/40.md) (Expiration Timestamp)
**Related**: [NIP-AZ](./NIP-AZ.md) (AltZap), [NIP-CZ](./NIP-CZ.md) (Cash Zaps)

## Abstract

This NIP defines two event kinds — Kind `35521` (Identity Connection) and Kind `35522` (IA Attestation) — that let a user cryptographically bind their Nostr public key to an account on a non-Nostr platform (a **Web Identity**: Discord, X, email, ...), witnessed by a permissionless, federated network of **Identity Authorities** (IAs). A deterministic verification-state model lets any client resolve a ConnectionKey — the hash NIP-AZ, NIP-CZ and NIP-CASH use to address a recipient who has no Nostr keypair yet — back into a verified Nostr identity, without trusting the claim on its own.

## Motivation

A self-signed claim of "this Nostr key is mine, and I also own this Discord account" is worthless on its own — anyone can sign that claim for any account they don't own. Something has to have actually checked the claim before publishing it.

This NIP splits the claim into two signed layers instead of one:

1. **The user's own claim** (Kind 35521) — cheap, replaceable, and can be freely re-published, but by itself proves nothing.
2. **A third party's witnessed proof** (Kind 35522) — a signature from an Identity Authority that actually performed the verification (checked a public post, sent an OTP, etc.) and is willing to stake its own reputation on the result.

Splitting these apart — rather than having a single self-attested event — means the *verification logic* (challenge issuance, evidence checking) is decoupled from the *identity graph* (which pubkey claims which account). Multiple independent IAs can attest the same connection without coordinating with each other, and a client only needs to trust **one** of them.

## Terminology

- **IA (Identity Authority)**: A service that verifies a user's ownership of a Web Identity account and signs a Kind `35522` attestation. Anyone can run one — there is no central gatekeeper.
- **Web Identity**: A non-Nostr identity platform (Discord, Telegram, X, GitHub, Instagram, Facebook, a DNS domain, or email). See [NIP-AZ](./NIP-AZ.md) for the shared Web Identity/ConnectionKey terminology.
- **ConnectionKey**: `hex(SHA256("<platform>:<platform-user-id>"))`, in lowercase hex — the hash NIP-AZ, NIP-CZ and NIP-CASH use. `<platform-user-id>` MUST be the canonical form given in §Canonical Platform User IDs. The `d` tag of both event kinds in this NIP.
- **Deep Check**: Fetching the Kind `35522` attestation from its origin relay (rather than trusting an unverified reference) to confirm it is genuinely signed, unexpired, and unrevoked.
- **Challenge Token (`npv1`)**: A bech32-encoded, session-bound proof string a user publishes publicly on the Web Identity platform as evidence of control.

## Specification

### Canonical Platform User IDs

Every client hashing a ConnectionKey for the same account MUST get the same hash, so `<platform-user-id>` has one canonical form per platform:

| Platform | `<platform>` | `<platform-user-id>` |
|---|---|---|
| X | `x` | The numeric account ID, in decimal. Not the handle. |
| Discord | `discord` | The numeric account ID, in decimal. Not the username. |
| Email | `email` | The whole address, trimmed and lowercased, its domain part as IDNA A-labels. |
| DNS domain | `dns` | The domain as IDNA A-labels, lowercased, without a trailing dot. |

No canonical form is defined here for other platforms.

A domain, the `dns` value or the part of an `email` after its last `@`, is written as IDNA A-labels (punycode, e.g. `xn--bcher-kva.example`), never as Unicode. Lowercasing is ASCII-only: only `A`–`Z` change. Trimmed means leading and trailing ASCII whitespace removed.

No value carries a leading `@`. A handle can be changed, then taken by someone else; an account ID cannot, so a ConnectionKey built from it keeps naming the same account.

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
    ["platform", "<platform-name>"]
  ],
  "content": "<public-profile-json>",
  "id": "...",
  "sig": "..."
}
```

Published by the end user to claim a connection. On its own this is an unverified claim — see Verification Model.

**Required tags**: `d` (the bare ConnectionKey — **MUST NOT** carry a platform prefix like `discord:`; a prefixed `d` tag breaks `#d` filter queries from third-party clients), `platform` (the Web Identity's platform name, for relay-side filtering), `e` (references the witnessing `35522`; see below).

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
    ["platform", "<platform-name>"],
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

**Required tags**: `d` (same ConnectionKey as the `35521` it will be referenced by), `p` (the Nostr pubkey that proved ownership — a client MUST verify this equals the outer `35521`'s `pubkey` when cross-checking, to prevent an attestation issued to one user being replayed inside a different user's `35521`), `platform`, `evidence` (see below). `expiration` is a [NIP-40](https://github.com/nostr-protocol/nips/blob/master/40.md) timestamp; a default of 90 days is RECOMMENDED, and an IA MAY omit it for a non-expiring attestation.

#### The `evidence` Tag

A cleartext JSON (v1) string:

| Field | Type | Description |
|---|---|---|
| `version` | number | Always `1`. |
| `platform` | string | Web Identity platform name. |
| `auth_type` | string | How the IA verified control of the account. `"public_post"` is defined below; IAs MAY use other values (e.g. a one-time code sent to an email address). |
| `user_id` | string | The platform account identifier, in its canonical form (§Canonical Platform User IDs). |
| `username` | string | The provider handle at verification time. |
| `verified_at` | number | Unix timestamp of successful verification. |
| `evidence_url` | string | `public_post` only: URL of the public post carrying the challenge. |
| `challenge` | string | `public_post` only: the `npv11...` challenge token published at `evidence_url`. |
| `pre_auth_code` | string | `public_post` only: the raw code the challenge was derived from — REQUIRED for cross-IA re-verification (see below). |

With `public_post`, verification is backed by a public, independently-fetchable URL, so any IA can re-verify the same claim without user interaction: fetch `evidence_url` and confirm `challenge` is present.

With any other `auth_type` (e.g. an email or phone handshake), there is no public evidence to re-check. The attestation rests only on the IA's signature, and a client that trusts the IA accepts that the IA completed the handshake. Such an attestation cannot be re-verified by another IA.

#### The `npv1` Challenge Token

A bech32-encoded TLV binding a challenge to one specific user and session, preventing replay across sessions:

```
type=0x00 || length=0x20 || value=SHA256(pubkey_bytes || pre_auth_code_bytes)
```

`pubkey_bytes` is the 32-byte raw public key (the hex pubkey, decoded). `pre_auth_code_bytes` is the UTF-8 encoding of the `pre_auth_code` string exactly as it appears in `evidence`. The bech32 human-readable part is `npv1`, so encoded tokens start with `npv11`.

A second IA performing cross-IA re-verification from existing evidence MUST:
1. Bech32-decode `challenge` to its 34-byte TLV payload.
2. Strip the 2-byte header (`0x00 0x20`) to recover the 32-byte session hash.
3. Recompute `SHA256(pubkey_bytes || pre_auth_code_bytes)` from the attestation's `p` tag and `evidence.pre_auth_code`.
4. Assert the two hashes are equal.

Step 4 is what distinguishes a genuinely session-bound challenge from a token replayed out of a different session — which is why `public_post` evidence MUST include `pre_auth_code`; without it, step 3 cannot be performed.

### The `nconnection` Bech32 Encoding

A portable, shareable string (e.g. handed from a Discord bot to a mobile wallet) bundling a ConnectionKey with the relays where its `35521` events are published, structured as TLV — the same pattern [NIP-19](https://github.com/nostr-protocol/nips/blob/master/19.md) uses for `nprofile`/`nevent`:

| Type | Name | Length | Description |
|---|---|---|---|
| `0` | Special | 32 bytes | The raw ConnectionKey bytes (hex-decoded). |
| `1` | Relay | variable | A relay URL, UTF-8. Repeatable. |

`nconnection` is **not** a registered NIP-19 prefix — it is a distinct, protocol-specific encoding that happens to reuse NIP-19's TLV convention for consistency.

## Verification Model

A client holding a 3-element identity tag (e.g. `["p", "<ConnectionKey>", "discord"]`, per NIP-AZ) resolves it to a Nostr identity as follows:

The IA is the authority, not the user's claim. An IA holds at most one current attestation per ConnectionKey: its latest `35522` for that `d` tag, which replaces any earlier one (see Event Lifecycle). The pubkey in that attestation's `p` tag is the only pubkey the IA links to the account. A `35521` that points to any other attestation is not verified, even if the attestation it points to is validly signed.

A client MUST NOT show a connection as **Verified** unless every check in step 3 passes. Steps 4 and 5 are MUST for a client that displays a state at all; how it renders each state is a SHOULD (see the table).

1. Query `{"kinds": [35521], "#d": ["<ConnectionKey>"]}`.
2. If no event is returned, the connection is **Unverified**: fall back to the Web Identity's own profile.
3. For each returned `35521` and each of its `e` tags, fetch the referenced `35522` from the relay hint (a Deep Check). The attestation counts only if all of the following hold:
   - it is a validly signed kind `35522` whose `pubkey` is an IA the client trusts;
   - its `d` and `platform` tags equal the `35521`'s `d` and `platform` tags;
   - its `p` tag equals the `35521`'s `pubkey`;
   - it is unexpired (NIP-40) and not deleted (NIP-09);
   - it is the IA's current attestation: querying the IA's relay for `{"kinds": [35522], "authors": ["<ia-pubkey>"], "#d": ["<ConnectionKey>"]}` returns this same event ID.
4. A `35521` with at least one attestation that counts is **Verified**. A `35521` without one is **Spoofed**: someone published a claim that does not check out, which is distinct from *Unverified*.
5. If trusted IAs' current attestations name different pubkeys for the same ConnectionKey, the attestation with the latest `created_at` wins; claims from every other pubkey are **Spoofed**.

| State | Condition | Recommended UI |
|---|---|---|
| **Verified** | `35521` found; it points to a trusted IA's current attestation for its own pubkey | ✅ native Nostr profile |
| **Unverified** | No `35521` found | ⏳ fallback Web Identity profile |
| **Spoofed** | `35521` found, but no attestation it points to passes step 3, or a newer attestation names another pubkey | ⚠️ flag as suspicious |

## Event Lifecycle

1. **Creation**: the user completes a verification flow with an IA; the IA signs `35522`.
2. **Publishing**: the IA publishes `35522` to its own relay(s).
3. **Linking**: the IA hands the `35522` event to the user's client, which builds the `e` tag from its event ID and the IA relay URL and publishes `35521`. The attestation itself is never embedded.
4. **Re-issue and transfer**: when the IA verifies the same account again, for the same pubkey or a new one, it publishes a new `35522` with the same `d` tag, which replaces the earlier one. Claims pointing to the replaced attestation stop verifying; the current owner publishes a new `35521` pointing to the new attestation.
5. **Revocation**: to revoke, the IA publishes a [Kind 5](https://github.com/nostr-protocol/nips/blob/master/09.md) deletion targeting the `35522` ID. This immediately fails any subsequent Deep Check.

## Federated Trust Model & Multi-IA Stacking

The IA layer is federated, not centralized: anyone can run an IA, and different wallets/clients choose which IAs they trust — there is no protocol-level allowlist. A user can strengthen their identity by having *multiple* independent IAs attest the same Web Identity account, referencing all of them via multiple `e` tags on one `35521`. A client only needs to trust one attesting IA to consider the connection verified, which gives resilience against any single IA going offline or being compromised.

## Security Considerations

- **Content-field spoofing**: `35521`'s `content` is user-controlled and unverified. An LNURL server MUST route payments strictly by the cryptographic `d`-tag hash, never by `content` — a user cannot redirect funds by editing their display name. A verifying client MUST cross-check `content` claims against the fetched `35522`'s `evidence` and flag a mismatch as spoofed, rather than rendering `content` at face value.
- **Self-attestation**: a `35522` is only meaningful because it's signed by a key distinct from the user's own — a verifier MUST confirm the attesting `pubkey` is a key it (or its trust set) recognizes as an IA, not the user's own key.
- **Cross-session replay**: the `npv1` challenge binds a specific `pubkey` + `pre_auth_code` pair (see above); without checking that binding, a challenge published for one verification session could be replayed to justify an unrelated one.
- **`p`-tag confusion attack**: an attacker could copy a real `35522` (issued to user A) and reference it from a `35521` published by user B. A verifying client MUST check that the attestation's `p` tag equals the outer `35521` event's own `pubkey` — a mismatch means the attestation does not belong to this connection, regardless of how it was referenced.
- **`d`-tag substitution attack**: user B verifies their own account X, then publishes a `35521` whose `d` is victim Y's ConnectionKey but whose `e` points to B's attestation for X. The `p` check passes, because that attestation really was issued to B. A verifying client MUST check that the attestation's `d` and `platform` equal the `35521`'s; otherwise B could claim Y's account and receive payments addressed to it.
- **Stale attestations**: a replaced or transferred `35522` can still be served by relays other than the IA's. A verifying client MUST confirm the referenced attestation is the IA's current one for that ConnectionKey (Verification Model, step 3), not just that it exists somewhere.
- **No private mode**: `evidence.user_id`/`evidence.username` are always cleartext and public once a `35522` is published. There is no way to link an identity without disclosing these fields on the relay.
- **Attestation expiry**: an IA SHOULD default to a bounded `expiration` (90 days is RECOMMENDED) rather than an unexpiring attestation, so a compromised or abandoned IA's attestations naturally age out even if the IA never explicitly revokes them.
