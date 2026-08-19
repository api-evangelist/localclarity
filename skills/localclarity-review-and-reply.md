---
name: Retrieve reviews and publish a reply
description: Pull reviews for a LocalClarity profile or location and post a public reply to one of them on Google or Facebook, without duplicating the reply on retry.
api: openapi/localclarity-openapi.yml
operations:
  - getProfiles
  - getReviews
  - sendReply
generated: '2026-08-13'
method: generated
source: openapi/localclarity-openapi.yml
---

# Retrieve reviews and publish a reply

`sendReply` is the only write operation LocalClarity's API exposes, and it publishes text that
the public will see on Google or Facebook. Read this whole file before calling it.

## Before you start

- API key in the `Authorization` request header, generated at **Reporting → Data Studio → API**.
- You need a `profileId` from `getProfiles` — see the *List profiles, accounts and locations*
  skill.

## Steps

1. **`getReviews`** — `POST /api/getReviews` with `profileId`, and optionally `locationId` to
   narrow to one business location. Returns an array of
   `{name, reviewId, reviewer{displayName,isAnonymous}, starRating, comment, createTime,
   updateTime, reviewReply{comment,updateTime}}`.
2. **Decide whether a reply is needed.** If `reviewReply` is already populated the review has a
   response. There is no documented "update reply" operation, so calling `sendReply` on an
   already-answered review is not a defined update path — check `reviewReply` first.
3. **`sendReply`** — `POST /api/sendReply`. All six parameters are documented as required:
   - `profileId` — the profile.
   - `reviewId` — from `getReviews`.
   - `source` — the review platform, e.g. `google` or `facebook`.
   - `locationId` — the Google location id, used when `source` is Google.
   - `pageId` — the Facebook page id, used when `source` is Facebook.
   - `reply` — the reply text.
   Returns `{replyId, reply, reviewId, reviewDocId, profileId, accountId, userId, source,
   date, time, postTime, replyStatus, googleUpdated}`.
4. **Confirm publication.** `replyStatus` and `googleUpdated` carry the delivery state.
   LocalClarity's platform performs a follow-up check with Google to confirm whether a reply
   was published, remains pending, or was rejected (release note, 2026-06-25), so a `200` from
   `sendReply` is an accepted-for-delivery signal, not proof of publication.

## Rules

- **There is no idempotency key.** This is the single most important constraint in this API.
  If `sendReply` times out you cannot safely retry it — a retry can publish a second public
  reply. Instead, re-run `getReviews` for that `locationId`, look at `reviewReply` on the
  target review, and only retry if it is still empty. Keep your own client-side record of
  `(reviewId, reply)` attempts.
- **`locationId` and `pageId` are source-dependent.** The apiDoc marks both as required but
  describes `locationId` as the Google location and `pageId` as the Facebook page. Send the
  one that matches `source`.
- **Errors.** `401` `{"message": "Unauthorized : ..."}` for a missing, invalid or revoked key;
  `403` `{"error": "Quota exceeded"}` for quota — not `429`, no `Retry-After`.
- **No pagination on `getReviews`.** The full array is returned for the requested scope.
- **Treat the reply text as public and permanent.** Nothing in the API undoes a published
  reply.
