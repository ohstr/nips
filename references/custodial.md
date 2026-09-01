# Custodial Logic & Fallback Addresses

A core differentiating mechanic of AltZap (NIP-AZ) is its capacity to route value to users who do **not** have a Nostr keypair yet (Legacy Identity Providers / LIDPs).

## The "Pending" State (Fallback Escrow)
When a ZSP (Zap Settlement Provider) settles a zap intended for a legacy user (e.g., an email address or Discord ID):

1. **Check for Discovery Link:** The ZSP checks if a **Kind 35521 (Identity Connection)** exists that bind that connection key/hash to an active Nostr pubkey.
2. **Fallback Holding:** If NO such link exists, the ZSP **cannot** successfully forward the funds because there is no endpoint. 
3. The ZSP MUST hold the funds in an internal ledger known as the "Fallback Address."

## The Universal AltZap Receipt (Finality)
Even when funds fall into a "Custodial" or "Pending" state where the recipient hasn't received them on-chain:
- **The ZSP MUST publish the Kind 5521 AltZap Receipt** to the network.
- *Why?* This informs the network (and the Sender) that the value has successfully moved from the Sender's wallet into the ZSP's secure escrow. From the perspective of the Sender, the payment is mathematically "Final".

## The Claim Flow Mandate
To ensure that funds are never permanently captive:
- Any compliant ZSP **MUST** provide a "Claim Flow" (e.g., a bot-session or challenge-token verification flow).
- When the LIDP user authenticates and generates a Nostr keypair, they go through the full verification flow:
  1. IA operator approves → Kind 35522 attestation published (session `confirmed`).
  2. User signs and publishes Kind 35521 referencing the attestation via `e` tag (event ID + relay URL — no embedded JSON).
  3. Client sends the activation signal to the IA — **only at this point** does the IA write the `Identity` record to its store. The mechanism for this signal is IA-implementation-defined.
- The ZSP detects the active `Identity` record and sweeps the custodial ledger to the newly bound Nostr wallet.

> **Important:** Publishing Kind 35521 alone is insufficient to activate routing. The ZSP routing table is only updated after the IA receives and processes the activation signal. A user who closes the signing dialog without completing activation has no active link and funds remain in escrow.
