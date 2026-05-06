# Flixbnb — Learnings

Lessons learned, gotchas, and non-obvious decisions made during development.

---

## Backend (`control-room`)

### SQS + Lambda Event Processing
- **Manual retry cap is a DLQ substitute**: both `handle-events` and `handle-agent-events` check `ApproximateReceiveCount > 3` to avoid infinite re-queuing. This works but silently drops messages — real DLQs should replace it.
- **FIFO queues require a `MessageGroupId`**: all SQS sends must include one or messages are rejected. We group by `reservationPmsId` or `listingId` to preserve ordering per entity.
- **Lambda `visibilityTimeoutSeconds` must exceed function timeout**: both are set to 300s. If the Lambda runs longer, SQS will re-deliver the message to another consumer while the first is still running.

### Lambda + Async: Don't Fire-and-Forget
- **AWS Lambda freezes Node.js when the handler resolves** — any un-awaited promises are killed. `applyCheckSuggestion` was called without `await` inside `handleAgentIncoming`, so the Lambda returned and AWS froze the process before the message was sent. This caused `RC_GUEST_WELCOME` and `RC_VERIFY_MESSAGE` to silently never send. Always `await` async side-effects in Lambda handlers before returning.

### AI / Agent Suggestions
- **Context size is the main cost driver**: `FrontDeskAgentTask` receives the full suggestion context even though it only uses part of it. Reducing context = lower latency + cost.
- **`bufferAllowWithoutHuman` bypasses the approval step**: when set on a `ReservationCheck`, the system auto-applies the AI suggestion without staff review. Only safe for low-risk check types (`RC_VERIFY_MESSAGE`, `RC_GUEST_WELCOME`).
- **Per-check-type suggestors**: each `ReservationCheckType` has its own suggestor class (e.g. `RcGuestWelcome`, `RcGuestCheckout`). Adding a new check type requires a new suggestor + registering it in `CheckSuggestionCreator`.
- **Model instructions are per operation mode**, not per check type: `backoffice-guest-operations`, `backoffice-staff-operations`, `frontdesk`, and `reminder` each have their own system prompt file. The suggestor picks the right one.
- **Suggestion group (`suggestionGroupId`)** ties all action options for a single check evaluation together. Staff approve one action from the group; the rest are discarded.

### Database
- **`pmsId` is a compound key**, not a raw integer: format is `"hostaway:accountid:listingid"` for listings and `"hostaway:accountid:reservationid"` for reservations. Never treat it as numeric.
- **`reservation_assignments.day` is stored as `varchar("2025/05/25")`**, not a proper `date` column. All date comparisons must parse this string — a known tech debt item.
- **`control_room` Postgres schema**: all tables live in the `control_room` schema, not `public`. Drizzle queries must use `controlRoomSchema.table(...)` or they'll hit the wrong schema.
- **Drizzle migrations are append-only**: never edit existing migration files in `/drizzle`. Always generate new ones with `npm run drizzle-generate`.

### Hostaway Integration
- **Reservations are synced, not fetched live**: vendor-net pipelines Hostaway data into Postgres on a schedule. The control-room API reads from DB, not Hostaway directly.
- **`channelId` determines messaging channel**: Airbnb reservations need OTA messages (via Hostaway); direct bookings use WhatsApp (WATI).

### Infra / Deployment
- **Two Docker images per deployment**: `Dockerfile` for the ECS Fargate service, `Dockerfile_lambda` for the Lambda image. Both built from the same `control-room` source.
- **`DEPLOY_CTRL_ROOM_SERVICE` / `DEPLOY_CTRL_ROOM_LAMBDA` env flags** control whether Pulumi rebuilds the Docker images. Omitting them skips the image build (useful for infra-only changes).
- **Secrets come from SSM at runtime**, not baked into the image. The ECS task role and Lambda role both have `ssm:GetParameter` permissions scoped to the relevant parameter paths.
- **Traceloop API key is currently hardcoded** in `src/utils/traceloop.ts` — should be moved to SSM/env before this becomes a problem.

---

## Control Platform (`flixbnb-lobby`)

### Data Fetching
- **All API calls go through `useGetData` / `useFindData` / `usePostData`** wrappers in `src/api/index.ts`. Don't use `fetch` directly — the wrappers handle auth headers, base URL, and error normalization.
- **Most views poll every 10–30 seconds** via `ahooks/useRequest` `refreshInterval`. This is intentional — the platform is used by multiple staff simultaneously and needs to stay live.
- **4XX responses from the backend are not always JSON** — the lobby API client has a workaround but error handling is inconsistent. When adding new error cases, test the raw response shape.

### Routing
- **`[partnerName]` vs `[partnerId]`**: public-facing pages (cleaners, guests) use `partnerName` in the URL; internal staff pages use `partnerId`. Don't mix them.
- **`[contacthash]`** is an opaque hash, not the DB `id`. Used in URLs sent to cleaners via WhatsApp so they can't enumerate other contacts.

### i18n
- **Cleaner-facing pages are translated** (EN/ES/EL); staff-facing pages are English only. Only add `next-intl` keys for pages cleaners or guests will see.

---

## General

### Partners
- Partners are the top-level multi-tenant unit. Almost every DB query and API call is scoped by `partnerId`. Missing a partner filter is the most common source of data leaks between partners.
- **Partner ID 1 (Ran)** is the primary test partner and has the most listings. Use it for local development.

### Messaging
- **WhatsApp session vs template messages**: WATI requires a pre-approved template for the first message to a guest. Once the guest replies, a 24-hour session window opens for free-form messages. The lobby composer (`WatiTemplateCompose` vs `WatiSessionMessageComposer`) handles this distinction.
- **Hostaway message threads are per-reservation**, not per-guest. A guest with two reservations has two separate message threads.

---

*Last updated: 2026-05-06*
