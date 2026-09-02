---
name: zapf-nips
description: Use when building, debugging, or verifying components of AltZap / NIP-AZ (Kinds 5520, 5521, 5523). Invoke for LNURL webhooks, ZSP implementations, proxy agents, BOLT11 validations, and identity resolution.
license: MIT
metadata:
  version: "2026.1.0"
  domain: protocol
  triggers: Zapf, AltZap, NIP-AZ, Kind 5520, Kind 5521, Kind 5523, ZSP, proxy zap, fallback address, custodial receipt
  role: specialist
  scope: architecture
  output-format: markdown
---

# NIP-AZ (AltZap) Core SKILL

AltZap (NIP-AZ) is a Non-Custodial Intent Layer built on Nostr and Lightning/Flokicoin. It bridges Legacy Identity Providers (LIDPs) to cryptographic value by decoupling Intent (Requests) from Proof of Settlement (Receipts).

## Role Definition

You are a strict, protocol-compliant ZSP (AltZap Settlement Provider) engineer. You strictly enforce cryptographic constraints, identity resolution, and correct Event formatting. Precision in tags (`p`, `P`, `amount`, `chain`) is mandatory.

## When to Use This Skill

- Whenever validating or creating an AltZap Request (Kind 5520) or On-Behalf AltZap Request (Kind 5523).
- When settling invoices and publishing AltZap Receipts (Kind 5521).
- When bridging Legacy Identities (Discord, Email, X) via Identity Connections (Kind 35521).
- When reasoning about custodial or fallback funds.

## Core Directives

1. **Verify the Identity Tag Array Structure**: `["p/P", "<hex_or_ConnectionKey>", "<lidp_name>"]` is the standard format for identity mapping (Kind 5521 AltZap Receipts can include an optional 4th element: `<handle>`).
2. **Obey Chain Gating**: A requested `chain` format must strictly match the network's liquidity capabilities.
3. **Mandate Fallbacks**: Unresolved Nostr identities must result in Custodial ZSP states until claimed.
4. **Clean #d Tags**: When querying or constructing `Identity Connection (35521)` filters with `#d`, the value MUST ONLY be the ConnectionKey. It must NEVER include the platform prefix (e.g., `["discord:<ConnectionKey>"]` is invalid; `["<ConnectionKey>"]` is correct).
5. **Nostr is NOT an LIDP**: Native Nostr identifiers (NIP-05 addresses like `bob@primal.net` or raw pubkeys) must **never** be deterministically hashed into ConnectionKeys. Only true legacy platforms (Discord, Email, X) use ConnectionKeys.
6. **Default Provider Mapping**: When querying the database for a fallback or local identity (which may have an empty `provider: ""` string internally), the API layer MUST default to `"nostr"` as the provider string in JSON responses. Do not use `"zapf"` or other labels.

## Reference Guide

Load the detailed architectural logic based on the context of the user's prompt:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| Event Kinds | `references/kinds.md` | Writing schemas, JSON structures, or determining which Kind to use (5520 vs 5523 vs 5521 vs 35521). |
| Validation | `references/validation.md` | Implementing `description_hash` binding, checking signatures, or Chain Gating logic. |
| Custodial | `references/custodial.md` | Dealing with unmatched external users, fallback addresses, or claim flows. |
| LNURL/LUD-16 | `references/lnurl.md` | Integrating LUD-06/LUD-16 resolution, Multi-Chain (Flokicoin) metadata, and parsing `/api/lnurl/cb` requests. |
| Identity Connection | `references/identity-connection.md` | Building Kind 35521/35522, ConnectionKey derivation, evidence format, `nconnection` bech32 encoding, or cross-IA challenge binding. For the full UI+backend session flow, see `../connection-lidp-flow/SKILL.md`. |

## Knowledge Base Quick-Sheet

- **LIDP**: Legacy Identity Provider (e.g., discord, email). **Nostr is NOT an LIDP.**
- **ZSP**: AltZap Settlement Provider (liquidity and receipt issuer).
- **Milli-Units**: `mloki` (Flokicoin) or `msats` (Bitcoin). Valid denomination. No base units.
- **ConnectionKey**: The unique SHA256 cryptographic hash used in the `p` or `P` tags when a Nostr pubkey is unknown (e.g., for LIDPs). Never hash native Nostr addresses.
