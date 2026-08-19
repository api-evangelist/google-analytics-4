---
name: Configure a GA4 property's custom dimensions and key events
description: Add custom dimensions, custom metrics and key events to a Google Analytics 4 property through the Admin API, safely, on an API that has no idempotency key.
api: openapi/google-analytics-4-admin-v1beta-openapi.yml
operations:
  - accountSummaries_list
  - properties_get
  - properties_customDimensions_list
  - properties_customDimensions_create
  - properties_customDimensions_patch
  - properties_customDimensions_archive
  - properties_customMetrics_list
  - properties_customMetrics_create
  - properties_keyEvents_list
  - properties_keyEvents_create
generated: '2026-08-13'
method: generated
source: Grounded in operationIds verified against openapi/google-analytics-4-admin-v1beta-openapi.yml
---

# Configure a GA4 property's custom dimensions and key events

## Before you start

- Host: `https://analyticsadmin.googleapis.com`.
- Scope: `https://www.googleapis.com/auth/analytics.edit`. The read-only scope will not do.
- **This API has no idempotency key.** There is no `Idempotency-Key` header, no client
  `requestId`, no `validateOnly`, no `dryRun`, and no `etag`. A create that times out may or may
  not have succeeded, and repeating it can produce a duplicate. Every write in this skill is
  therefore preceded by a read.

## Steps

1. **Resolve the property.** `accountSummaries_list` →
   `GET /v1beta/accountSummaries`, then optionally `properties_get` →
   `GET /v1beta/properties/{propertiesId}` to confirm you have the right one. Carry the full
   `properties/{propertyId}` resource name forward.

2. **Read before you write.** `properties_customDimensions_list` →
   `GET /v1beta/properties/{propertiesId}/customDimensions`. Page with `pageSize` / `pageToken`
   until `nextPageToken` is absent. Match on `parameterName` and `displayName` to decide whether
   the dimension you want already exists. Do the same with
   `properties_customMetrics_list` and `properties_keyEvents_list`.

3. **Create only what is missing.** `properties_customDimensions_create` →
   `POST /v1beta/properties/{propertiesId}/customDimensions`. Custom dimension slots are finite
   per property, so treat creation as a scarce, irreversible-ish action.
   Same shape for `properties_customMetrics_create` and `properties_keyEvents_create`.

4. **Update with an explicit field mask.** `properties_customDimensions_patch` →
   `PATCH /v1beta/properties/{propertiesId}/customDimensions/{customDimensionsId}`.
   You **must** send `updateMask` naming the fields you are changing. Omitting it is an error.
   Sending `updateMask=*` replaces the whole resource — only do that deliberately.

5. **Retire rather than delete.** `properties_customDimensions_archive` →
   `POST /v1beta/properties/{propertiesId}/customDimensions/{customDimensionsId}:archive`.
   There is no delete for custom dimensions or custom metrics. Archiving frees the slot; it does
   not remove historical data.

## Rules

- **Use `keyEvents`, not `conversionEvents`.** `properties_conversionEvents_*` are flagged
  `deprecated: true` in Google's own Discovery document, superseded by `properties_keyEvents_*`.
  The reporting side made the same rename: `isConversionEvent` became `isKeyEvent` and the
  `conversions` metrics became `keyEvents` on 2024-05-06.
- **Respect the write quota.** The Admin API allows 600 writes per minute per project and 180 per
  minute per user, inside an overall 1,200 requests per minute. There are no rate-limit headers to
  tell you where you are; count your own calls.
- **Errors:** `{"error": {"code": …, "status": "CANONICAL_CODE"}}`. `403 / PERMISSION_DENIED`
  usually means either the principal has no role on the property or
  `analyticsadmin.googleapis.com` is not enabled on the calling Cloud project — check both.
- **Do not blind-retry a create.** On a timeout, re-list and match by `displayName` before trying
  again.

## Related

- Access control lives on `properties/{property}/accessBindings` and
  `accounts/{account}/accessBindings`, which are **v1alpha only** —
  `openapi/google-analytics-4-admin-v1alpha-openapi.yml`. They are the only GA4 resources with
  batch operations.
- Data model: `data-model/google-analytics-4-data-model.yml`
- Conventions: `conventions/google-analytics-4-conventions.yml`
