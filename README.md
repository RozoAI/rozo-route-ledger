# Rozo route ledger

`route-ledger.json` is a public, aggregated snapshot of which payment routes work for Rozo checkout. A route is one combination of merchant, source chain, source token and amount bucket. The file tells a wallet, an agent or a checkout page which routes are supported and, where there is enough volume, how reliable they have been recently.

It is regenerated every hour from Rozo's internal route outcome records and committed to this repository.

## What is in it, and what is not

The snapshot is built for publication, so it is deliberately coarse:

* Merchants are named only when they are already listed in the public checkout catalog, https://checkout.rozo.ai/services.json. Every other route is published as part of an anonymous `other` bucket, see below.
* No wallet addresses, no transaction hashes, no payment links, no hashed user or device keys.
* No counts. Volume never appears, only whether a route crossed the publication threshold.
* Success rate is published only as one of three buckets: `>=95%`, `80-95%`, `<80%`.
* Time is published at ISO week granularity only (for example `2026-W41`).
* Statistics appear for a route only when it had at least 30 terminal outcomes in the last 14 days. Below that, a route only says whether it is supported.
* Internal and test traffic is excluded.

Historical success is a preference signal, not a guarantee. A consumer must still check the current quote, fees, expiry and the payer's balance before paying.

## URLs

* Raw (freshest): https://raw.githubusercontent.com/RozoAI/rozo-route-ledger/main/route-ledger.json
* jsDelivr CDN: https://cdn.jsdelivr.net/gh/RozoAI/rozo-route-ledger@main/route-ledger.json

The publisher asks jsDelivr to purge its cache after every push, but CDN caching can still lag. Consumers that care about freshness should read the raw URL, and every consumer must apply the staleness rule below.

## Staleness rule

Consumers must treat the ledger as absent when `generated_at` is more than 6 hours older than the consumer's current time. With no ledger, a consumer must not show or act on any recommendation and must behave exactly as it would without this file.

`generated_at` advances on every hourly run even when no route changed, so a stale timestamp always means the publisher stopped, never that nothing happened.

## Schema

Top level object:

| Field | Type | Meaning |
| --- | --- | --- |
| `ledger_version` | integer | Schema version. Currently `1`. A consumer that does not know the version should ignore the file. |
| `generated_at` | string, ISO 8601 UTC | When this snapshot was computed. Drives the staleness rule. |
| `window_days` | integer | Window for the published statistics, in days (currently 14). |
| `thresholds` | object | The gates a route must pass to be `recommended`, see below. |
| `routes` | array | One entry per route, sorted by merchant, chain, token, amount bucket. |

`thresholds` object:

| Field | Meaning |
| --- | --- |
| `min_terminal_events` | Minimum terminal outcomes in the window before a route can be recommended. |
| `min_success_rate` | Minimum share of terminal outcomes that completed. |
| `max_p90_receive_minutes` | Maximum 90th percentile time from order creation to funds received on the source chain. |
| `max_route_failure_rate_7d` | Maximum share of paid orders in the last 7 days that ended mismatched or refunded. |
| `max_days_since_completed` | The route must have completed a payment within this many days. |
| `min_ledger_age_days` | The ledger must have been recording live for this many days before anything is recommended. |

Each entry in `routes`:

| Field | Type | Meaning |
| --- | --- | --- |
| `merchant_app_id` | string | Either a catalog merchant, written `merchant_<slug>` where `<slug>` is the `slug` field of an entry in services.json (for example `merchant_openrouter`), or the literal `other`. |
| `source_chain_id` | string or null | Chain the payer pays from: an EVM chain id such as `8453`, a Rozo chain code such as `1500`, or `lightning`. |
| `source_token` | string or null | Token symbol the payer pays with, for example `USDC`. |
| `amount_bucket` | string or null | Order size bucket in USD: `lt1`, `1to10`, `10to50`, `50to200`, `200to1000`, `gte1000`. |
| `supported` | boolean | True when this route completed at least one payment in the last 30 days. |
| `success_rate_bucket` | string | Only present above the publication threshold. Share of terminal outcomes in the window that completed, as `>=95%`, `80-95%` or `<80%`. Expired quotes count as not completed. |
| `recommended` | boolean | Only present above the publication threshold. True when the route passes every gate in `thresholds`. |
| `last_completed_week` | string | Only present above the publication threshold. ISO week of the most recent completed payment. |

### Merchant names and the `other` bucket

services.json is a list of services keyed by `slug`; it has no app id field. Rozo checkout uses `merchant_<slug>` as the merchant app id, so the allowlist is exactly the set of `merchant_<slug>` values derived from the catalog. The publisher refreshes its copy of the catalog at most once every 24 hours, keeps using its last good copy if a refresh fails, and publishes nothing if it has never obtained one.

Routes of every other merchant or app are combined into one entry with `merchant_app_id` set to `other` for each combination of `source_chain_id`, `source_token` and `amount_bucket`. Their terminal and completed counts are added together first, and only then are the publication threshold and the success rate buckets applied. `supported` is true when any combined route completed a payment in the last 30 days. An `other` entry is never `recommended`, because the gates cannot be combined across merchants and nobody can choose an anonymous mix.

Fields may be added in a later version without changing `ledger_version`; consumers should ignore fields they do not know.

## How it is produced

An internal job mirrors Rozo's route outcome records read only, computes the view, filters it for publication, and refuses to publish if the output contains anything shaped like a wallet address or a hash. The job commits here only when the content changed or when `generated_at` is due to advance, which is every hour.

## License

The data is published by Rozo for anyone to read and use.
