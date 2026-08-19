---
name: List LocalClarity profiles, accounts and locations
description: Walk the LocalClarity identifier chain from an API key to the business locations it can reach, so later calls have the profileId, accountId and locationId they require.
api: openapi/localclarity-openapi.yml
operations:
  - getProfiles
  - getOrganizations
  - getLocations
generated: '2026-08-13'
method: generated
source: openapi/localclarity-openapi.yml
---

# List LocalClarity profiles, accounts and locations

Every LocalClarity call except `getProfiles` requires a `profileId`. Start here before any
review or insights work.

## Before you start

- You need an API key. An administrator generates it in the LocalClarity app at
  **Reporting → Data Studio → API → Generate New Key**. The full key is displayed once.
- Send it on every request as the `Authorization` request header. There is no scheme prefix
  documented — send the token value.
- Base host: `https://dev.localclarity.com`. LocalClarity's published apiDoc declares
  `https://localclarity.cloud.tyk.io`, but that hostname did not resolve when probed on
  2026-08-13. Confirm the host with support@localclarity.com before building against it.

## Steps

1. **`getProfiles`** — `GET /api/getProfiles`. No parameters. Returns an array of
   `{role, profileName, userId, profileId}`. Keep `profileId`; it is mandatory everywhere else.
2. **`getOrganizations`** — `POST /api/getOrganizations` with `profileId`. Returns an array of
   `{accountId, accountName, userId}`. Use `accountId` when you want to narrow locations to a
   single account.
3. **`getLocations`** — `POST /api/getLocations` with `profileId`, and optionally `accountId`.
   Returns an array of Google Business Profile location resources. The fields you will most
   often key off are:
   - `name` — Google's identifier, in the form `accounts/{account_id}/locations/{location_id}`.
     The trailing segment is the `locationId` other operations expect.
   - `storeCode` — the customer's own external identifier, unique within an account. Use this
     to join LocalClarity data to internal records.
   - `locationName`, `address`, `primaryCategory`, `regularHours`, `openInfo`, `locationState`.

## Rules

- **Do not page.** No pagination parameter, cursor or link header exists. These endpoints
  return the full array, so size the request timeout for the largest profile in the portfolio.
- **These reads are POST.** `getOrganizations` and `getLocations` are POST despite being reads,
  so they are not cacheable and not safe to replay blindly through a proxy.
- **Handle two error shapes.** A missing, invalid or revoked key returns `401` with
  `{"message": "Unauthorized : ..."}`. Quota exhaustion returns `403` with
  `{"error": "Quota exceeded"}` — not `429`, and with no `Retry-After`. Back off on your own
  schedule; see `errors/localclarity-problem-types.yml`.
- **Revocation is immediate.** If a key is revoked in Data Studio, in-flight requests fail
  with `401`. Treat a sudden 401 on a previously working key as revocation, not as a transient.
