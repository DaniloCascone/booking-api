# Spec 001: Create booking

Status: draft
Owner: <your name>

## Goal
An authenticated user can book a resource for a time interval. The operation is safe to retry, safe under concurrency, and rate limited.

## Context
- Users and resources already exist (assume tables `users` and `resources`; create minimal versions if missing).
- Authentication is out of scope for this spec: the user id comes from a validated JWT `sub` claim. In tests, use a helper that injects a user id.

## API contract
`POST /api/v1/bookings`

Headers:
- `Authorization: Bearer <jwt>` (required)
- `Idempotency-Key: <string, 1-64 chars>` (required)

Body:
```json
{ "resourceId": "uuid", "startTime": "2026-12-01T10:00:00Z", "endTime": "2026-12-01T11:00:00Z" }
```

Responses:
- `201 Created` with the booking (`id`, `resourceId`, `userId`, `startTime`, `endTime`, `createdAt`)
- `400` invalid body or missing Idempotency-Key
- `401` missing or invalid token
- `404` resource not found
- `409` the interval overlaps an existing booking for that resource
- `422` same Idempotency-Key reused with a different body
- `429` rate limit exceeded, with `Retry-After` header

## Constraints
1. Times are stored as UTC `timestamptz`. Intervals are half-open: `[start, end)`, so back-to-back bookings are allowed.
2. `startTime` must be in the future and strictly before `endTime`. Maximum duration: 4 hours.
3. Overlap prevention is enforced by PostgreSQL (`EXCLUDE USING gist (resource_id WITH =, tstzrange(start_time, end_time) WITH &&)`, requires `btree_gist`). The service translates the violation into `409`.
4. Idempotency: a repeated request with the same user, same key and same body returns the original response (same status and body) and does not create a second booking. Same key with a different body returns `422`. Keys are scoped per user.
5. Rate limit: maximum 10 requests per minute per user id, using Bucket4j (in memory is fine for now). A replay of an already processed idempotency key does NOT consume the limit.
6. Metrics (Micrometer): `bookings.created`, `bookings.conflict`, `bookings.rate_limited` counters.

## Acceptance criteria
1. Valid request returns `201` and the booking is persisted.
2. Missing `Idempotency-Key` returns `400`.
3. `endTime <= startTime`, past `startTime`, or duration over 4 hours returns `400`.
4. Unknown `resourceId` returns `404`.
5. Overlapping interval on the same resource returns `409`; the same interval on a different resource succeeds.
6. Back-to-back intervals (`end` of one = `start` of the other) both succeed.
7. Two concurrent requests for the same slot (different users): exactly one gets `201`, the other `409`. Covered by an integration test with real threads.
8. Same key + same body sent twice: both responses are identical, only one row exists.
9. Same key + different body returns `422`.
10. The 11th request within one minute by the same user returns `429` with `Retry-After`; another user is not affected.
11. Counters are incremented in the matching scenarios (verified in tests).

## Out of scope
Cancelling or updating bookings, payments, notifications, admin endpoints, multi-instance rate limiting (Redis), authentication implementation.

## Open questions
- Should the rate limit count requests rejected with `400`? (Default: yes, they are still requests.)
- How long are idempotency keys kept? (Default: 24 hours, cleanup job out of scope.)
