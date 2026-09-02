# LNURL & LUD-16 Integration

AltZap (NIP-AZ) uses **LUD-06** and **LUD-16** to bridge traditional Lightning wallets to Nostr infrastructure and Web Identities via AltZap Settlement Providers (ZSP).

When modifying, debugging, or creating LNURL flows within Zapf, you **MUST** adhere to the following strict integration rules.

## 1. Identifier Resolution (The `username`)

When Zapf receives a request at `GET /.well-known/lnurlp/<username>`, the `<username>` isn't necessarily a standard handle. It MUST be resolved in this exact priority order:

1. **NIP-05 Handle** (`alice` in `alice@domain.com`): Looked up natively by querying the target domain's `.well-known/nostr.json`. 
2. **ConnectionKey Hash** (`a1b2c3...`): A 64-character hex string representing a legacy social connection (e.g. Discord). **NEVER** expose the raw Web Identity username (like `discorduser`) in the LNURL path.
3. **Raw Pubkey** (`<hex_pubkey>`): Used directly as the Nostr recipient.

### Custodial Fallback (The Proxy Behavior)

If the identifier is completely unknown (e.g. a random string or unmapped Discord ID), the ZSP **MUST STILL RETURN A VALID LNURL RESPONSE**.
- The ZSP operates as a catch-all escrow (Proxy).
- The `nostrPubkey` returned in the proxy metadata is the ZSP's master server pubkey.
- At the callback phase, an invoice is generated for the ZSP's master wallet, and the funds are held custodially until the user claims the handle.

## 2. Chain Metadata Gating

Zapf extends the standard LUD-16 metadata array to support multi-chain parameters (e.g., Flokicoin).

- If the service operates on Flokicoin, the metadata array MUST include the tuple: `["chain/flokicoin", "loki"]`.
- The `minSendable` and `maxSendable` values are denominated in the **smallest sub-unit** of the declared chain (e.g., `milliloki` for Flokicoin, or `msats` for Bitcoin).

**Example compliant metadata array:**
```json
[
  ["text/plain", "Zap a1b2c3...@zapf.network"],
  ["text/identifier", "a1b2c3...@zapf.network"],
  ["chain/flokicoin", "loki"]
]
```

*Note on Proxy Endpoints: If no chain tags are present in the LNURL metadata (`/.well-known/lnurlp/proxyUsername`), clients default to `chain/flokicoin`.*

## 3. The Callback & Cryptographic Verification

When the wallet requests the invoice:
`GET /api/lnurl/cb/<username>?amount=<milliunits>&nostr=<signed_event>&comment=<text>`

The backend MUST enforce these checks:
1. **Amount Assertion**: If a `nostr` event (Kind 5520, 5523) is attached, extract the `amount` tag. It MUST exactly match the `amount` query parameter requested by the wallet. If there is a mismatch, reject the request with a 400 Error.
2. **Chain Assertion**: The frontend generating the `nostr` AltZap Request EVENT must explicitly include the `["chain", "flokicoin"]` tag in the event to prevent cross-chain replay attacks.
3. **Description Hash**: 
   - If a `nostr` event is provided, the invoice's description hash MUST be the SHA256 hash of the raw JSON string of the Nostr event.
   - If no `nostr` event is provided (a plain Lightning payment), the description hash MUST be the SHA256 hash of the LNURL `metadata` JSON array.

## 4. Error Handling (Browser Frontend)

When fetching the callback URL from standard web clients:
- Standard Lightning nodes often lack permissive CORS headers (`Access-Control-Allow-Origin: *`).
- Browsers will forcibly block the callback request, throwing a generic `TypeError: Failed to fetch`.
- Frontend code MUST catch this generic error and gracefully assume it is a CORS, Mixed-Content (HTTPS invoking HTTP), or Offline Node issue, rather than crashing or displaying unhelpful stack traces.
