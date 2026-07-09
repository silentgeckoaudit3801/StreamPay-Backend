# Accrual, Webhook Delivery, and Settlement Semantics

This page documents the behavior implemented in the current backend code. It is intentionally limited to repository behavior that can be verified from `src/` today.

## Accrual Preview Semantics

Implemented in: `src/services/accrualService.ts` (`AccrualService.calculateAccrual`).

The accrual preview calculates an estimated amount since the stream's last settlement checkpoint:

```text
accrued = ratePerSecond * max(0, min(now, endTime) - lastSettledAt)
```

The service derives each term from the `Stream` row in `src/db/schema.ts`:

| Term | Source |
| --- | --- |
| `ratePerSecond` | `streams.rate_per_second`, declared as `decimal(20, 9)` |
| `lastSettledAt` | `streams.last_settled_at` |
| `endTime` | `streams.end_time`, when present |
| `now` | optional method argument, defaulting to `new Date()` |

Behavior details:

- If `stream.status !== "active"`, the service returns `"0.000000000"` and does not accrue further.
- If an `endTime` exists and `now > endTime`, the calculation stops at `endTime`.
- Negative elapsed time is clamped to zero with `Math.max(0, ...)`.
- The result is formatted with `toFixed(9)`, matching the 9-decimal scale used by `decimal(20, 9)` stream amount/rate columns.
- The returned `AccrualResult` includes `streamId`, `accruedAmount`, `calculationTimestamp`, and the current stream `status`.

This is a preview calculation. It does not write a settlement checkpoint, transfer funds, or mutate stream state.

## Outbound Webhook Delivery Lifecycle

Implemented in: `src/services/webhookDeliveryService.ts` and `src/db/schema.ts`.

Outbound delivery records are stored in the `webhook_deliveries` table:

| Column | Meaning |
| --- | --- |
| `subscription_id` | Subscription that should receive the event |
| `event_type` | Delivered StreamPay event type |
| `payload` | JSON string sent to the subscriber |
| `status` | `pending`, `success`, or `failed` |
| `attempts` | Number of delivery attempts already made |
| `next_attempt_at` | Earliest time the worker should retry |
| `last_http_status` | HTTP status returned by the last attempt, if any |
| `last_error` | Error message from the last attempt, if any |

Lifecycle:

1. `WebhookDeliveryService.enqueue(event)` finds enabled subscriptions matching `event.eventType` and creates one `pending` delivery per subscription with `attempts = 0` and `nextAttemptAt = new Date()`.
2. `processDue()` asks the repository for due pending deliveries and calls `attempt()` for each one.
3. `attempt()` loads the subscription. If it no longer exists, the delivery is marked `failed` with `lastError = "Subscription not found"`.
4. For a live subscription, the service signs the raw JSON payload with HMAC-SHA256 using `signPayload()`, then POSTs it with `X-StreamPay-Signature` and `X-StreamPay-Event` headers. The request timeout is 10 seconds.
5. Any `response.ok` result marks the delivery `success`, increments `attempts`, and stores `lastHttpStatus`.
6. Non-2xx HTTP responses and thrown fetch errors record either `HTTP <status>` or the thrown error message.
7. Failed deliveries with `attempts < MAX_ATTEMPTS` stay `pending`, update `attempts`, `lastHttpStatus`, `lastError`, and schedule `nextAttemptAt` with exponential backoff.
8. Failed deliveries with `attempts >= MAX_ATTEMPTS` become permanently `failed`.

Retry constants:

| Symbol | Value | Source |
| --- | --- | --- |
| `MAX_ATTEMPTS` | `5` | `webhookDeliveryService.ts` |
| `BASE_DELAY_MS` | `5_000` | `webhookDeliveryService.ts` |
| `MAX_DELAY_MS` | `300_000` | `webhookDeliveryService.ts` |

The retry delay is `BASE_DELAY_MS * 2^(attempt - 1)`, capped at `MAX_DELAY_MS`.

## Indexer Webhook Ingestion Semantics

Implemented in: `src/routes/webhooks/indexer.ts` and `src/services/eventIngestionService.ts`.

`POST /webhooks/indexer` requires API key authentication before raw body parsing. The route then requires the raw JSON body because signature verification uses the exact bytes from the request.

Response mapping in the route handler:

| Condition | HTTP status | Body shape |
| --- | --- | --- |
| First accepted event | `200` | `{ accepted: true, duplicate: false, eventId, eventType }` |
| Duplicate accepted event | `202` | `{ accepted: true, duplicate: true, eventId, eventType }` |
| Invalid signature | `401` | `{ error: "invalid_signature", message }` |
| Invalid JSON | `400` | `{ error: "invalid_json", message }` |
| Invalid payload | `400` | `{ error: "invalid_payload", message }` |
| Missing server secret | `500` | `{ error: "missing_secret", message }` |
| Request body was not a Buffer | `400` | `{ error: "invalid_body", message }` |

The duplicate path is a safe no-op: it reports acceptance with `202` rather than reprocessing the event.

## Settlement Flow Status

`docs/ARCHITECTURE.md` previously described a `Settlement Service` in the data-flow diagram. That service is planned, but it is not implemented as a `src/services/settlementService.ts` module today.

Current settlement-adjacent code is split across:

- `src/services/accrualService.ts` for read-only accrual previews.
- `src/services/transactionService.ts` for transaction-oriented operations.
- `src/services/eventIngestionService.ts` for trusted indexer events.
- `src/services/webhookDeliveryService.ts` for outbound subscriber notifications.
- `src/clients/sorobanClient.ts` for Soroban client interactions.

Until a dedicated settlement service exists, contributors should avoid treating the architecture diagram as a complete implementation map. Settlement documentation should cite concrete code paths and database fields, as this page does.