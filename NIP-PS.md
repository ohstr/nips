NIP-PS
======

Profile Stats: Zap, Follow and Identity Assertions
--------------------------------------------------

`draft` `optional`

**Depends on**: [NIP-85](https://github.com/nostr-protocol/nips/blob/master/85.md) (Trusted Assertions), [NIP-57](https://github.com/nostr-protocol/nips/blob/master/57.md) (Lightning Zaps), [NIP-AZ](./NIP-AZ.md) (AltZap)

## Abstract

A provider of NIP-85 user assertions (kind `30382`) counts a profile's zaps and follows by the rules below, so the numbers can't be inflated by forged receipts, self-zaps or replayed payments, and adds tags for what NIP-85 leaves out: following count, distinct zappers, first and last zap times, and verified NIP-05 and lightning addresses.

## Motivation

Zap totals are easy to fake. A zap receipt is a signed event like any other: anyone can publish a kind `9735` naming any recipient and any amount, a recipient can zap themselves, and one payment can be published as several receipts. A provider that sums receipts as they come publishes numbers that reward exactly that. These rules make the totals mean what a client shows them as: money other people paid.

## Assertion tags

A kind `30382` assertion about a user carries NIP-85's `d` tag (the user's pubkey) and any of these tags. Values are strings; amounts are in whole units (sats for Bitcoin); times are unix seconds.

| Tag | NIP-85 | Value |
|-----|--------|-------|
| `rank` | yes | 0–100 |
| `followers` | yes | profiles whose latest contact list (kind `3`) has a `p` tag for this user |
| `following` | — | distinct valid pubkeys in this user's latest contact list, the user excluded |
| `zap_cnt_recd`, `zap_amt_recd` | yes | zaps received that count (see below) and their total |
| `zap_cnt_sent`, `zap_amt_sent` | yes | zaps sent that count, and their total |
| `zappers_recd` | — | distinct senders among the zaps received |
| `zap_first_recd`, `zap_last_recd` | — | `created_at` of the earliest and latest zap received |
| `zap_first_sent`, `zap_last_sent` | — | the same for zaps sent |
| `nip05_verified` | — | the user's NIP-05 identifier, present only while the provider has verified it |
| `lud16_verified` | — | the user's lightning address (`lud16`, or `lud06` when there is none), present only while the provider has verified its LNURL-pay endpoint |

A provider SHOULD omit a time tag that has no value and SHOULD keep an assertion's `created_at` unchanged while its tags are unchanged, so the event id stays the same.

## Counting zaps

A zap receipt (kind `9735`, or `9521` per NIP-AZ) counts for its recipient and its sender only if all of these hold:

1. It validates: NIP-57 Appendix F for `9735`, NIP-AZ's verification algorithm for `9521` (embedded request signature, invoice binding, preimage where required).
2. Once the provider has fetched the recipient's LNURL-pay endpoint and it advertises a `nostrPubkey`, the receipt is signed by that key. A provider that doesn't know the key yet MAY count a receipt that passes (1) and SHOULD recount when it learns it.
3. Its sender is not its recipient.
4. No other counted receipt for the same recipient has the same payment hash.
5. It hasn't been deleted (NIP-09) by its signer.

The **recipient** is the receipt's resolved pubkey (NIP-AZ `r`) when present, else its `p` tag when that names a Nostr pubkey; a receipt for an unresolved web identity credits no one. The **sender** is NIP-AZ's resolved sender (`R`), else a Nostr `P` tag, else the signer of the embedded request — except a NIP-AZ on-behalf request (`9523`), whose signer is a proxy and whose `P` names the sender. The **amount** is the paid invoice's amount, falling back to the receipt's or the request's `amount` tag only when the invoice has none.

## Rank

How `rank` is computed is the provider's own. A provider that ranks partly on zaps received SHOULD weigh each sender's contribution by how established the sender is and cap what one sender can add, so zaps from many fresh keys, or from a recipient's own LNURL server paying itself, move the rank little.

## Verification

`nip05_verified` and `lud16_verified` reflect fetches the provider made. A provider SHOULD keep a result until it expires (days, not minutes), re-fetch when the identifier changes, and treat a timeout, `403`, `429` or `5xx` as "try again later" rather than "not verified": dropping a verified tag because a server was briefly unreachable misinforms every client that reads it.

## Querying

Clients ask a provider (found through kind `10040`, or a relay that lists NIP-85 in its NIP-11 document) for assertions by target:

```json
["REQ", "s", {"kinds": [30382], "#d": ["<pubkey>", "<pubkey>"]}]
```

A provider relay MAY compute assertions when asked instead of storing them, and MAY refuse a request with no `#d` or too many targets with `CLOSED`.
