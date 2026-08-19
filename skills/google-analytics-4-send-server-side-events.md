---
name: Send server-side events to GA4 with the Measurement Protocol
description: Provision a Measurement Protocol secret through the Admin API and send validated server-side events to Google Analytics 4, on a fire-and-forget endpoint that never reports its own errors.
api: openapi/google-analytics-4-admin-v1beta-openapi.yml
operations:
  - properties_dataStreams_list
  - properties_dataStreams_get
  - properties_dataStreams_measurementProtocolSecrets_list
  - properties_dataStreams_measurementProtocolSecrets_create
generated: '2026-08-13'
method: generated
source: >-
  Grounded in operationIds verified against openapi/google-analytics-4-admin-v1beta-openapi.yml;
  Measurement Protocol behaviour from
  https://developers.google.com/analytics/devguides/collection/protocol/ga4 and
  https://developers.google.com/analytics/devguides/collection/protocol/ga4/validating-events
---

# Send server-side events to GA4 with the Measurement Protocol

The Measurement Protocol is the one part of GA4 that does not use OAuth and does not tell you when
you are wrong. `POST /mp/collect` returns **204 with an empty body** whether the event was perfect
or garbage. Validation is a separate endpoint and it is not optional in practice.

## Steps

1. **Find the data stream.** `properties_dataStreams_list` →
   `GET https://analyticsadmin.googleapis.com/v1beta/properties/{propertiesId}/dataStreams`.
   Pick the web or app stream you are instrumenting; `properties_dataStreams_get` returns its
   detail including the Measurement ID (`G-XXXXXXXXXX` for web streams).

2. **Check for an existing secret before minting one.**
   `properties_dataStreams_measurementProtocolSecrets_list` →
   `GET /v1beta/properties/{propertiesId}/dataStreams/{dataStreamsId}/measurementProtocolSecrets`.
   These are real credentials and there is no idempotency key on the create — a blind retry mints
   a second live secret you then have to notice and revoke.

3. **Create one only if needed.**
   `properties_dataStreams_measurementProtocolSecrets_create` →
   `POST /v1beta/properties/{propertiesId}/dataStreams/{dataStreamsId}/measurementProtocolSecrets`.
   Requires the `analytics.edit` scope. The returned `secretValue` is the `api_secret` below.
   Store it as a secret; it is not retrievable in plaintext from the GA4 UI afterwards.

4. **Validate the event first.**
   `POST https://www.google-analytics.com/_debug_/mp/collect?measurement_id=G-XXXXXXXXXX&api_secret=<secret>`
   (EU: `https://region1.google-analytics.com/_debug_/mp/collect`).
   A clean event returns `{"validationMessages": []}`. Anything else returns entries carrying
   `fieldPath`, `description` and a `validationCode` such as `NAME_INVALID`, `VALUE_REQUIRED` or
   `EXCEEDED_MAX_ENTITIES`. Events sent here are discarded and never appear in reports.

5. **Send it for real.**
   `POST https://www.google-analytics.com/mp/collect?measurement_id=G-XXXXXXXXXX&api_secret=<secret>`
   (EU: `https://region1.google-analytics.com/mp/collect`).
   Expect 204 and no body.

## Rules

- **The validation server does not check `api_secret` or `firebase_app_id`.** A request can
  validate perfectly and still be dropped in production because the credential is wrong. Verify
  those independently — confirm the event lands in DebugView or a realtime report.
- **Verify delivery out of band.** Because collect is fire-and-forget, the only proof an event
  arrived is `properties_runRealtimeReport` on the Data API
  (`openapi/google-analytics-4-data-v1beta-openapi.yml`) or DebugView in the console.
- **Never put the api_secret in client-side code.** It is a server-side credential; anyone holding
  it can inject events into the property.
- **No documented rate limit.** Google publishes event and parameter size caps for the
  Measurement Protocol but no request rate. Absence of a published limit is not permission to
  flood it.
- App streams use `firebase_app_id` instead of `measurement_id`.

## Related

- Sandbox and validation tooling: `sandbox/google-analytics-4-sandbox.yml`
- Errors and validation codes: `errors/google-analytics-4-problem-types.yml`
- Client-side collection (gtag.js): `components/google-analytics-4-components.yml`
