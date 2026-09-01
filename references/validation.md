# Validation & Cryptographic Enforcement rules

A Zap Settlement Provider (ZSP) relies on strict logic to protect users from malicious operators or cross-network unit loss.

## Description Hash Binding (Kinds 5520 & 5523)
For LNURL-Pay generated invoices, the ZSP MUST ensure that the `description_hash` (the `h` tag) in the generated BOLT11 invoice mathematically binds the exact AltZap Request event.

`BOLT11.h = SHA256(JSON.stringify(EventKind_5520_or_5523))`

If the hashes misalign, the payment is unprovable on the Nostr network, and the client must reject the attempt.

## Chain Gating Protocol
Every single request (`5520`, `5522`, `5523`) MUST define its network context to prevent "cross-chain collisions" (e.g., sending Milliloki but the receiver wanted Millisats, resulting in a dramatic loss or gain of underlying fiat value).

1. Perform LNURL-PAY GET request to the target destination.
2. Inspect the metadata array for a `chain/<name>` entry.
3. If the user's AltZap Request contains `["chain", "flokicoin"]`, but the endpoint's metadata only returns `chain/bitcoin`, the ZSP MUST REJECT the payment.

## Receipt Mirroring (Kind 5521)
To maintain logical, deterministic indexing for Nostr clients filtering via tags (`#p` or `#P`):
- The ZSP MUST blindly copy the `P` tag from the Request intent into the Receipt.
- The ZSP MUST blindly copy the `p` tag from the Request intent into the Receipt (unless it is a `5522` knowledge-less request, where the ZSP has the option to append it post-settlement via internal logs/claim links).
