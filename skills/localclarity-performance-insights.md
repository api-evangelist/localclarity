---
name: Pull Google Business Profile performance insights
description: Retrieve dated Google Business Profile performance metrics for a LocalClarity profile or a single location, for reporting and BI pipelines.
api: openapi/localclarity-openapi.yml
operations:
  - getProfiles
  - getLocations
  - getInsights
generated: '2026-08-13'
method: generated
source: openapi/localclarity-openapi.yml
---

# Pull Google Business Profile performance insights

This is the flow behind the use case LocalClarity names in its own key-generation article:
feeding listing and review data into a BI tool, reporting dashboard or internal warehouse.

## Before you start

- API key in the `Authorization` request header. LocalClarity recommends **one dedicated key
  per integration** so it can be audited and revoked independently — name it after the
  consumer, e.g. `Production – Looker`.
- Store the key in an environment variable or secret store, never in source control. Per-key
  request audit logs are retained for 12 months.

## Steps

1. **`getProfiles`** — `GET /api/getProfiles`. Capture `profileId`.
2. **`getLocations`** (optional) — `POST /api/getLocations` with `profileId`. Use it to build
   the `locationId` → `storeCode` map that joins LocalClarity metrics to your internal
   location records. `storeCode` is the customer-supplied external identifier and is the right
   join key for a warehouse.
3. **`getInsights`** — `POST /api/getInsights` with `profileId`, and optionally `locationId`
   to scope to a single location. Returns an array of
   `{date, metric, count, locationId, locationName, address, timeZone}` — one row per metric
   per date, which loads directly into a fact table.

## Rules

- **No date-range parameter is documented.** `getInsights` accepts only `profileId` and an
  optional `locationId`. Filter by the returned `date` field on your side, and do not assume a
  window — measure what the response actually covers before scheduling incremental loads.
- **No pagination.** For a large portfolio, prefer looping per `locationId` over one
  profile-wide call so a single response stays bounded.
- **These are POST reads.** Do not expect caching, conditional requests or ETags.
- **Quota is enforced as `403` `{"error": "Quota exceeded"}`**, not `429`, with no
  `Retry-After`. A scheduled extract should treat 403 as retryable-with-backoff and alert if
  it persists — LocalClarity does not publish the threshold, so discover yours empirically or
  ask support@localclarity.com.
- **`401` means the key is gone.** Revocation is immediate and irreversible; a pipeline that
  starts returning 401 needs a new key, not a retry.
- **The metrics are Google's.** `metric` values and semantics come from Google Business
  Profile performance data surfaced through LocalClarity, not from a LocalClarity-native
  metric model.
