---
name: Run a Google Analytics 4 report
description: Discover a GA4 property, validate the dimension/metric combination, and run a core report through the Data API without burning quota on rejected requests.
api: openapi/google-analytics-4-data-v1beta-openapi.yml
operations:
  - accountSummaries_list
  - properties_getMetadata
  - properties_checkCompatibility
  - properties_runReport
generated: '2026-08-13'
method: generated
source: Grounded in operationIds verified against openapi/google-analytics-4-data-v1beta-openapi.yml and openapi/google-analytics-4-admin-v1beta-openapi.yml
---

# Run a Google Analytics 4 report

## Before you start

- Auth is Google OAuth 2.0. There are no API keys for this API. Send
  `Authorization: Bearer <access_token>`.
- Scope: `https://www.googleapis.com/auth/analytics.readonly` is sufficient for everything here.
- Two hosts are involved: `https://analyticsadmin.googleapis.com` for discovery and
  `https://analyticsdata.googleapis.com` for the report itself.
- Reporting is metered in **tokens**, not calls, and there is no way to compute a request's cost
  in advance. Validate before you spend.

## Steps

1. **Find the property.** `accountSummaries_list` —
   `GET https://analyticsadmin.googleapis.com/v1beta/accountSummaries`.
   Returns the account and property tree in one call. Page with `pageSize` / `pageToken`
   until `nextPageToken` is absent. Take the property's `property` field, which is already in
   `properties/{propertyId}` form. Do not strip it to a bare id — every downstream call wants the
   full resource name.

2. **Enumerate what this property can actually report on.** `properties_getMetadata` —
   `GET https://analyticsdata.googleapis.com/v1beta/properties/{propertiesId}/metadata`.
   Returns every valid dimension and metric name including the custom ones defined on this
   property. Never guess a dimension or metric name; a wrong name is a 400 / `INVALID_ARGUMENT`
   that still costs you a round trip.

3. **Check the combination is legal.** `properties_checkCompatibility` —
   `POST https://analyticsdata.googleapis.com/v1beta/properties/{propertiesId}:checkCompatibility`.
   GA4 rejects many dimension/metric pairings. This operation tells you which of your requested
   fields are incompatible before you pay the token cost of the real report. Run it whenever the
   combination is not one you have already proven.

4. **Run the report.** `properties_runReport` —
   `POST https://analyticsdata.googleapis.com/v1beta/properties/{propertiesId}:runReport`.
   Required in the body: at least one entry in `dateRanges`, at least one `dimensions` entry, at
   least one `metrics` entry. Maximum nine dimensions per request.
   Always set `"returnPropertyQuota": true` — it is the only way to read your remaining budget,
   because this API sends no rate-limit response headers.

5. **Page the results.** Default cap is 10,000 rows per response. Page with `limit` and `offset`
   in the request body. This surface does **not** use `pageToken`; that is the Admin API pattern.

## Rules

- **No idempotency.** This flow is read-only, so retries are safe here — but do not carry that
  assumption into the Admin API, which has no idempotency key at all.
- **Branch on `error.status`, never on `error.message`.** Errors arrive as
  `{"error": {"code": …, "message": …, "status": "CANONICAL_CODE"}}`. This is google.rpc.Status,
  not RFC 9457 problem+json.
- **Cap your retries.** 500 / `INTERNAL` and 503 / `UNAVAILABLE` warrant exponential backoff, but
  each one counts against the Server Errors Per Project Per Property Per Hour quota — 10 on a
  standard property, 50 on Analytics 360. Unbounded retries lock the property out entirely.
- **429 / `RESOURCE_EXHAUSTED` means back off and re-read the budget**, not retry immediately.
  Standard properties get 200,000 tokens per day and 40,000 per hour; a single project is further
  capped at 14,000 tokens per property per hour.
- Demographic dimensions (`userAgeBracket`, `userGender`, `brandingInterest`, `audienceId`,
  `audienceName`) are subject to thresholding and are limited to 120 potentially-thresholded
  requests per hour.

## Related

- `properties_runRealtimeReport` for the last 30 minutes of activity.
- `properties_runPivotReport` and `properties_batchRunReports` for pivoted and batched variants.
- `properties_runFunnelReport` exists only in v1alpha —
  `openapi/google-analytics-4-data-v1alpha-openapi.yml`.
- Conventions: `conventions/google-analytics-4-conventions.yml`
- Errors: `errors/google-analytics-4-problem-types.yml`
- Quotas: `rate-limits/google-analytics-4-rate-limits.yml`
