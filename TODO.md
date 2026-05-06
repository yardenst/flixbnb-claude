# Flixbnb — TODO

---

## Backend (`control-room`)

### Infrastructure
- [ ] **Set up SQS Dead Letter Queues** for both `events` and `agent-events` FIFO queues. Currently both `handle-events.ts` and `handle-agent-events.ts` have a manual 3-retry cap as a stopgap — this logic can be removed once DLQs are in place.
- [ ] **Move Traceloop API key to env var / SSM** — the key is currently hardcoded in `src/utils/traceloop.ts`. Should be loaded from `process.env` like other secrets.
- [ ] **Replace direct `triggerHandleListingAsync` calls in listing issues route** with an SQS event — marked as `// TODO: temp` in `src/routes/v1/listings/issues/index.ts`.

### AI / Agent Pipeline
- [x] **`RC_GUEST_WELCOME` and `RC_VERIFY_MESSAGE` auto-send was fire-and-forget** — `applyCheckSuggestion` was not awaited, so AWS Lambda froze Node.js before it completed. Fixed by awaiting the call (`src/checks/reservations/index.ts`).
- [ ] **Auto-apply suggestions on a separate queue** — `RC_VERIFY_MESSAGE` and `RC_GUEST_WELCOME` auto-approval currently runs inline on the agent queue (`src/checks/reservations/index.ts`). Should be dispatched to its own queue to avoid blocking.
- [ ] **Remove `GUEST_MESSAGE` action type from staff-ops model** once `ASK_STAFF` action messaging is implemented (`backoffice-staff-operations-model-instructions.ts:1`).
- [ ] **`FrontDeskAgentTask` context refactor** — task currently receives the full suggestion context; it should fetch only what it needs (`src/checks/reservations/check-suggestions/suggestors/frontdesk/tasks/FrontDeskAgentTask.ts:14–18`).
- [ ] **Per-agent prompt fetching** — agent tasks should fetch their own prompts from DB rather than receiving them from the caller (`FrontDeskAgentTask.ts:18`).
- [ ] **`NaiveCheckSuggestionCreator` partner system instructions** — hardcoded partner-level instructions should be moved to the DB prompts system (`NaiveCheckSuggestionCreator.ts:74`).
- [ ] **Add flixchat messages to `ListingIssueCheck` context** so the AI has full context when evaluating listing issues (`src/checks/listings/checks/ListingIssueCheck.ts:8`).
- [ ] **Add `getCleaningAssignmentOnReservationDepartureDate` to `RcCleanerConfirmed` suggestor context** (`suggestors/RcCleanerConfirmed.ts:23`).
- [ ] **Fix `CollectorCheckSuggestionCreator` responsibility assignment** — marked as broken (`collectors/CollectorCheckSuggestionCreator.ts:208`).

### Schema / Data
- [ ] **Convert `reservation_assignments.day` from `varchar` to `date` type** — format is `"2025/05/25"`, a proper date column would be cleaner (`db/schema.ts:260`).
- [ ] **Delete `preReservationAssignmentId` column** from the assignments view — marked `// TODO: delete` in `db/views_schema.ts:9`.
- [ ] **Fix `RC_GUEST_CONFIRMS_CHECKIN` visibility logic** — `check-list.ts:38` has an incomplete condition (needs to only surface the check 1 hour after `RC_GUEST_GETS_INFO` is done).

### Code Quality
- [ ] **Create DB init wrapper for Lambdas** — `trigger-events.ts`, `handle-events.ts`, `handle-agent-events.ts`, and `trigger-agent-events.ts` all call `initDb()` directly with a `// TODO: Create Wrapper` note.
- [ ] **Fix `contacts/index.ts` stats query** — fetches all `partner_contacts_listings` rows without a `WHERE` clause (`contacts/index.ts:93`). Should be filtered by partner.
- [ ] **Fix message display in `messages/index.ts`** — comment says the current format doesn't match what the human sees in the Hostaway web UI (`messages/index.ts:136`).
- [ ] **Verify phone ownership before sending messages** — `routes/v1/messages/index.ts:85` has an unresolved todo to check that the phone belongs to the reservation before sending.
- [ ] **Fix `assignments/index.ts` cleaning guest count bug** — incorrect guest count assignment when `nextReservation.arrivalDate === assignment.day` (`assignments/index.ts:157`).
- [ ] **Use real timezones in flixchat routes** — two places in `routes/v1/flixchat/index.ts` (lines 124, 159) use a placeholder instead of the listing's actual timezone.
- [ ] **Make backend 4XX responses always return JSON** — `flixbnb-lobby/src/api/index.ts:75` notes that the API inconsistently returns non-JSON for some error responses.
- [ ] **Remove leftover `reservationId` route comment** (`routes/v1/reservations/index.ts:334`).

---

## Control Platform (`flixbnb-lobby`)

### Features
- [x] **Add per-task translate to Spanish button** — added 🇪🇸 button to each task card in `ContactTasksDashboard.tsx`; toggles translated title/description via `/api/translate` Next.js route.
- [ ] **Add `agentSuggestedAt` to `ReservationCheck` type** — field is commented out in `src/types.ts:529`.
- [ ] **Fix timezone in guest chat** — `Chat.tsx:11` uses a hardcoded timezone instead of the listing's timezone.

### Error Handling
- [ ] **Improve error handling in `ChecksDashboard`** — three separate `//todo handle it better` in `ChecksDashboard.tsx` (lines 210, 309, 339).
- [ ] **Improve error handling in `ListingChecksDashboard`** — same issue in `ListingChecksDashboard.tsx` (lines 79, 105, 134).

---

## Landing Page (`flixbnb`)

- [ ] **Wire up remaining Firebase SDKs** — `App.tsx:7` has a placeholder comment from the Firebase setup wizard for additional SDK integrations (e.g., Firestore, Auth) if needed.

---

## Infrastructure / DevOps

- [ ] **Add Dead Letter Queue Lambdas** in `infra-services` to handle and alert on DLQ messages once DLQs are created (depends on the SQS DLQ task above).
- [ ] **Automate release tagging** — `RELEASE_TAG` env var must be set manually when deploying via Pulumi. Consider wiring it to CI/CD.

---

*Last updated: 2026-05-06*
