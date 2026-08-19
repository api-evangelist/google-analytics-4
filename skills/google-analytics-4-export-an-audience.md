---
name: Export a GA4 audience
description: Run a long-running Google Analytics 4 audience export end to end — create it, poll for completion, and read the rows — including the one webhook surface GA4 offers.
api: openapi/google-analytics-4-data-v1beta-openapi.yml
operations:
  - properties_audienceExports_create
  - properties_audienceExports_get
  - properties_audienceExports_list
  - properties_audienceExports_query
generated: '2026-08-13'
method: generated
source: Grounded in operationIds verified against openapi/google-analytics-4-data-v1beta-openapi.yml and openapi/google-analytics-4-data-v1alpha-openapi.yml
---

# Export a GA4 audience

Audience export is the only long-running operation in the stable GA4 Data API, and the only place
in the whole provider where GA4 will call you back instead of making you poll.

## Before you start

- Host: `https://analyticsdata.googleapis.com`.
- Scope: `https://www.googleapis.com/auth/analytics.readonly`.
- The audience itself is an Admin API resource (`properties/{property}/audiences`, v1alpha). This
  skill assumes it already exists and you have its resource name.

## Steps

1. **Kick off the export.** `properties_audienceExports_create` →
   `POST /v1beta/properties/{propertiesId}/audienceExports`.
   Body names the `audience` resource and the `dimensions` you want on each row. Returns a
   long-running operation immediately — the rows are not ready yet.

2. **Poll until it is ready.** `properties_audienceExports_get` →
   `GET /v1beta/properties/{propertiesId}/audienceExports/{audienceExportsId}`.
   Watch `state`. `percentageCompleted` gives progress and `rowCount` tells you how much you are
   about to read. Back off between polls — every poll is a Data API call against the same quota
   as a report.

3. **Read the rows.** `properties_audienceExports_query` →
   `POST /v1beta/properties/{propertiesId}/audienceExports/{audienceExportsId}:query`.
   Page with `limit` and `offset` in the body, the same as reporting — not `pageToken`.

4. **Find earlier exports.** `properties_audienceExports_list` →
   `GET /v1beta/properties/{propertiesId}/audienceExports`, paged with `pageSize` / `pageToken`.
   Check this before creating a new export: creation charges
   `creationQuotaTokensCharged` against your token budget, so re-using a finished export is
   materially cheaper than making another.

## Getting called back instead of polling

Use the v1alpha siblings — `properties_audienceLists_create` and
`properties_recurringAudienceLists_create` in
`openapi/google-analytics-4-data-v1alpha-openapi.yml`. Both accept a `webhookNotification` object:

- `uri` — HTTPS only, valid certificate, maximum 128 characters, RFC 1738 characters only.
- `channelToken` — an arbitrary string up to 64 characters, echoed back so you can verify the
  source.

Google POSTs the operation resource plus a `sentTimestamp` (unix microseconds) to your URI, and
expects **HTTP 200 within 5 seconds**. Requests carry a Google OIDC ID token for the service
account `google-analytics-audience-export@system.gserviceaccount.com`; on Cloud Run or Cloud
Functions grant it `roles/run.invoker` / `roles/cloudfunctions.invoker`, and on any other server
you may ignore the token and verify with `channelToken` instead.

## Rules

- **Replays happen and are yours to detect.** `sentTimestamp` is the only de-duplication signal.
  There is no delivery id, no HMAC signature, and no published retry policy.
- **The webhook path is alpha.** The stable v1beta audience-export surface has no callback at all;
  choosing webhooks means accepting v1alpha stability terms.
- **No idempotency key.** A retried create can produce a second export and charge you twice. List
  first.
- Errors are google.rpc.Status; branch on `error.status`.

## Related

- Webhook surface: `asyncapi/google-analytics-4-webhooks.yml`
- Quotas: `rate-limits/google-analytics-4-rate-limits.yml`
- Data model: `data-model/google-analytics-4-data-model.yml`
