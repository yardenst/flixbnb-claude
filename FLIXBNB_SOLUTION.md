# Flixbnb — Solution Architecture

## Overview

Flixbnb is a **vacation rental operations platform** that helps property managers automate and manage the full guest lifecycle — from inquiry to checkout — across listings on Airbnb and other OTAs. The system integrates with Hostaway (PMS) and WhatsApp (WATI) to centralize operations.

The product consists of two apps:

| App | Folder | Role |
|---|---|---|
| Marketing site | `flixbnb` | Public-facing landing page |
| Control platform | `flixbnblobby` | Internal ops dashboard for staff |

---

## 1. Marketing Site — `flixbnb`

### Purpose

A React SPA deployed to Firebase Hosting. Its sole job is to generate leads — it presents the value proposition, pricing, and a CTA that links to an external Typeform intake form.

### Tech Stack

| Layer | Technology |
|---|---|
| Framework | React 19 |
| Build tool | Vite 6 |
| Language | TypeScript 5.7 |
| Styling | Tailwind CSS 4 |
| Hosting | Firebase Hosting |
| Analytics | Firebase Analytics |

### Structure

```
flixbnb/
├── src/
│   ├── App.tsx          # Entire SPA — single long-form marketing page
│   ├── main.tsx         # React entry point
│   ├── App.css          # Global styles
│   └── assets/          # Images and static assets
├── public/
├── index.html
├── vite.config.ts
├── firebase.json        # Firebase Hosting config
└── .firebaserc          # Firebase project binding
```

### Key Details

- **Single component**: `App.tsx` renders the full landing page — hero, problem statement, services, pricing ($50/listing starting price), and CTA buttons.
- **Analytics**: Firebase Analytics tracks campaign parameters (`outreach`, `outrich`) for attributing signups.
- **Color scheme**: Orange (`#fd8f00`, `#ff8d00`) on black backgrounds.
- **Build**: `npm run build` → Vite outputs to `/dist` → `firebase deploy` publishes.

---

## 2. Control Platform — `flixbnblobby`

### Purpose

A Next.js 15 full-stack application used by internal Flixbnb staff to manage every aspect of vacation rental operations: reservation lifecycle checks, guest communications, cleaner scheduling, and property issues.

### Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript 5 |
| UI components | Ant Design 5 |
| Styling | Tailwind CSS 4 + antd-style |
| Data fetching | ahooks (`useRequest`) |
| Date/time | Luxon |
| Auth | next-auth 5 (beta) |
| i18n | next-intl 4 |
| Database client | Neon (serverless Postgres) |
| WhatsApp | WATI Business API |
| PMS | Hostaway |

### Directory Structure

```
flixbnblobby/
├── src/
│   ├── api/                          # API client hooks
│   │   ├── index.ts                  # Core hooks: useGetData, useFindData, usePostData, etc.
│   │   ├── partner.ts
│   │   ├── reservations.ts
│   │   ├── assignments.ts
│   │   ├── listing-checks.ts
│   │   └── listing-issues.ts
│   │
│   ├── app/                          # Next.js App Router
│   │   ├── layout.tsx                # Root layout (Ant Design, i18n)
│   │   ├── page.tsx                  # Home — partner selector
│   │   │
│   │   ├── [partnerName]/            # Public partner pages
│   │   │   └── Chats.tsx             # Guest chat interface
│   │   │
│   │   ├── api/                      # Next.js API routes
│   │   │   ├── proxy-image/          # Image proxy (CDN compat)
│   │   │   └── proxy-safari-video/   # Video proxy (Safari compat)
│   │   │
│   │   ├── chat/[reservationId]/     # Guest-facing chat
│   │   │
│   │   ├── cleaning-requests/[listinghash]/
│   │   │
│   │   ├── contacts/[contacthash]/
│   │   │   ├── cleanings/            # Cleaner's assigned cleanings
│   │   │   └── tasks/                # Cleaner's assigned tasks
│   │   │
│   │   ├── partners/[partnerId]/
│   │   │   ├── checks/               # Reservation lifecycle checks (core)
│   │   │   ├── assignments/          # Cleaner scheduling
│   │   │   ├── listings/             # Property management
│   │   │   │   └── checks/           # Property-specific checks
│   │   │   ├── instructions/         # Check instruction editor
│   │   │   ├── flixchat/             # Staff messaging
│   │   │   └── control/              # Admin oversight
│   │   │       ├── cleaners/
│   │   │       └── listings/
│   │   │
│   │   ├── prompts/                  # AI prompt template editor
│   │   └── wati/                     # WhatsApp components
│   │
│   ├── types.ts                      # Shared TypeScript interfaces
│   ├── listingChecks.ts              # Listing check types
│   ├── reservationChecks.ts          # Reservation check types
│   └── utils/datetime.ts
│
├── messages/
│   ├── en.json
│   ├── es.json
│   └── el.json
├── next.config.ts
├── tailwind.config.js
└── .env
```

---

## 3. Data Models

### Partner
The top-level organizational unit. Four active partners: Ran, Click & Keys, WaveStay, UrbanKey.

```typescript
{
  id: number
  name: string
  hostawayAccountId: number
  cleaningPublishDate: string
}
```

### FlixbnbListing
A property managed by Flixbnb. Contains operational documentation used to inform AI suggestions and staff instructions.

```typescript
{
  id: number
  partnerId: number
  pmsId: number
  nickname: string
  timezone: string
  isEnabled: boolean
  contactNumber?: string
  defaultNumberOfGuests: number

  // Operational docs (all string fields, freeform)
  checkInInstructions: string
  checkOutInstructions: string
  checkInManual: string
  houseRules: string
  amenitiesManual: string
  hospitalityManual: string
  cleaningProcedures: string
  maintenanceProcedures: string
  laundryProcedures: string
  essentialInfoForGuest: string
  knownIssues: string
  guestInfoRequirements: string
  contacts: string
  floorPlan: string

  reservationChecks: { checks: { type: ReservationCheckType }[] }
}
```

### HostawayReservation
Sourced from Hostaway PMS. Central to all operations.

```typescript
{
  id: number
  listingMapId: number
  listingName: string
  channelId: number
  channelName: string           // "Airbnb", etc.
  reservationId: string
  guestName: string
  guestFirstName: string
  guestLastName: string
  phone: string
  guestCountry: string
  numberOfGuests: number
  arrivalDate: string
  departureDate: string
  checkInTime: number
  checkOutTime: number
  nights: number
  status: string
  confirmationCode: string
  guestNote: string
  guestPortalUrl: string
  reservationAgreement: string
  accountId: number
}
```

### ReservationCheckType
19 checkpoint types that map the full guest journey and cleaning lifecycle:

| Check Type | Description |
|---|---|
| `RC_GUEST_INQUIRY` | Respond to guest inquiry |
| `RC_GUEST_WELCOME` | Send welcome message |
| `RC_GUEST_GETS_INFO` | Send check-in info packet |
| `RC_GUEST_DETAILS_CONFIRMED` | Collect guest arrival details |
| `RC_GUEST_CONFIRMS_CHECKIN` | Verify guest knows how to check in |
| `RC_GUEST_WILL_CHECKIN_FOR_SURE` | Confirm imminent arrival |
| `RC_GUEST_CHECKED_IN_WELL` | Verify successful check-in |
| `RC_GUEST_STAY_IS_GOOD` | Mid-stay satisfaction check |
| `RC_GUEST_CHECKOUT` | Checkout verification |
| `RC_GUEST_REVIEW` | Confirm guest left review |
| `RC_GUEST_REVIEW_COMMENT` | Reply to guest review |
| `RC_GUEST_PAID` | Charge outstanding balance |
| `RC_GUEST_CONFIRMS_TIMES` | Verify check-in time |
| `RC_CLEANER_ASSIGNED` | Assign cleaner to turnover |
| `RC_CLEANER_CONFIRMED` | Verify cleaner confirmed |
| `RC_LISTING_CLEAN_FOR_STAY` | Verify property is ready |
| `RC_VERIFY_MESSAGE` | Send a verification message |
| `RC_REMINDER` | Handle scheduled reminder |
| `RC_WHAT_WE_LEARNED` | Document learnings post-stay |

### ReservationCheck
An instance of one check type for one reservation.

```typescript
{
  id: number
  listingId: number
  reservationPmsId: string
  reservationCheckType: ReservationCheckType
  checkVisibleAt: string          // Scheduled visibility time

  // Completion
  checkedBy?: string
  checkedAt?: string
  checkedMethod?: string

  // Removal / snooze
  removedAt?: string
  snoozedAt?: string
  snoozeReason?: string

  // AI
  agentSuggestedAt?: string
  responsibleContactId?: number
  responsibleNotes?: string

  // Messaging
  reservationMetadata?: ReservationCheckVerifyMessageMetadata
  bufferAllowWithoutHuman?: boolean
  notificationResult?: string
}
```

### AgentSuggestion
AI-generated action recommendation for a check.

```typescript
{
  id: number
  reservationCheckId: number
  suggestionGroupId: number
  actionType: "DONE" | "INQUIRY" | "GUEST_MESSAGE" | "REMINDER" |
              "SNOOZE" | "GUEST_NUDGE" | "UPDATE_RECORDS" |
              "LISTING_ISSUE" | "VOID" | "ASK_STAFF"
  actionPayload: {
    message?: string
    urgency?: number
    snoozeAt?: Date
    label?: string
  }
  metadata: {
    reason: string
    bottomLineProof?: string
    approvedAt?: number
    urgencyReason?: string
    channel?: "ota" | "whatsapp" | "call"
    messagesToSend: { content: string, checkType: string }[]
  }
}
```

### CleaningAssignment

```typescript
{
  id: number
  day: string
  contactId: number             // Cleaner
  listingId: number
  priority: number
  assignmentType: 'clean'
  cleaningNumberOfGuests?: number
  cleaningCheckInTime?: number
  cleaningCheckOutTime?: number
  cleaningNotes?: string
  cleaningApprovedAt?: Date
}
```

### Contact
Cleaner or staff member.

```typescript
{
  id: number
  partnerId: number
  name: string
  phone: string                 // WhatsApp number
  job: "cleaner"
  note: string
}
```

### ListingIssue

```typescript
{
  id: number
  listingId: number
  description: string
  note?: string
  resolvedAt?: string
}
```

---

## 4. Backend API Reference

All requests go to `CONTROL_ROOM_BACKEND_URL` (env var). Base path: `/v1/`.

### Partners
| Method | Path | Description |
|---|---|---|
| GET | `/v1/partners` | All partners |
| GET | `/v1/partners/:partnerId` | Single partner |
| POST | `/v1/partners/:partnerId/cleanings` | Update cleaning publish date |
| GET | `/v1/partners/:partnerId/contacts` | Partner contacts |
| GET | `/v1/partners/:partnerId/allContacts` | All contacts incl. unassigned |

### Listings
| Method | Path | Description |
|---|---|---|
| GET | `/v1/listings/?partnerId=` | Partner listings |
| GET | `/v1/listings/issues` | All listing issues |
| POST | `/v1/listings/issues` | Create issue |
| PUT | `/v1/listings/issues` | Update issue |
| POST | `/v1/listings/issues/resolve` | Resolve issue |

### Reservations
| Method | Path | Description |
|---|---|---|
| GET | `/v1/reservations/ongoing?partnerId=` | Active reservations |
| GET | `/v1/reservations/search?phone=` | Search by phone |
| GET | `/v1/reservations/byPmsId?reservationPmsId=` | Single reservation |
| GET | `/v1/reservations/metadata?reservationPmsId=` | Reservation metadata |
| GET | `/v1/reservations/next?reservationPmsId=` | Next reservation for guest |
| GET | `/v1/reservations/prev?reservationPmsId=` | Previous reservation for guest |
| GET | `/v1/reservations/assignments-query?day=&daysInTheFuture=&partnerId=` | Reservations for assignment view |
| GET | `/v1/reservations/cleaning-before?reservationPmsId=` | Cleaning before this reservation |
| GET | `/v1/reservations/cleaning-after?reservationPmsId=` | Cleaning after this reservation |
| POST | `/v1/reservations/edit` | Update check-in/out times |

### Checks
| Method | Path | Description |
|---|---|---|
| GET | `/v1/checks/reservations/open?partnerId=` | Open checks |
| GET | `/v1/checks/reservations?partnerId=&reservationId=` | Checks for a reservation |
| GET | `/v1/checks/reservations/instructions?partnerId=` | Check instructions |
| GET | `/v1/checks/reservations/notes?reservationPmsId=` | Check notes |
| GET | `/v1/checks/reservations/responsible/:contactHash/open` | Open checks assigned to contact |
| GET | `/v1/checks/:checkId/suggestions` | AI suggestions for a check |
| POST | `/v1/checks/listings/done` | Mark listing check done |
| GET | `/v1/checks/checksTemplateMap` | WATI template mapping |

### Assignments (Cleaning)
| Method | Path | Description |
|---|---|---|
| GET | `/v1/assignments?partnerId=&type=clean&day=` | Cleaning assignments |
| GET | `/v1/assignments/contact?id=` | Assignments for contact |
| GET | `/v1/assignments/contact/cleanings?id=` | Cleaning jobs for contact |
| POST | `/v1/assignments/cleaning/edit` | Update assignment |
| POST | `/v1/assignments/cleaning/remove` | Remove assignment |
| POST | `/v1/assignments/cleanings/publish` | Publish to cleaners via WhatsApp |

### Messaging
| Method | Path | Description |
|---|---|---|
| GET | `/v1/messages/reservation?reservationPmsId=` | Conversation history |
| GET | `/v1/messages/wati/templates` | WhatsApp templates |

### Other
| Method | Path | Description |
|---|---|---|
| GET | `/v1/contacts/stats?partnerId=` | Contact activity stats |
| GET | `/v1/contact-reminders/?partnerId=` | Contact reminders |
| GET | `/v1/ai-suggestions/snapshots` | AI suggestion snapshots |
| GET | `/v1/prompts` | AI prompt templates |
| GET | `/v1/flixcontrol/stats?reservationsCount=&partnerId=` | Control room stats |
| POST | `/v1/flixcontrol/runFlixRating` | Run Flix rating calculation |
| GET | `/v1/reviews/find?reservationPmsId=` | Guest reviews |
| GET | `/v1/inquiries` | Inquiries |
| GET | `/v1/notifications` | Notifications |
| GET | `/v1/cleaning-requests/assignments-query?day=&daysInTheFuture=` | Cleaning requests |

---

## 5. Key Features

### Reservation Lifecycle Checks (Core Feature)

The `checks` module is the heart of the platform. Each reservation gets a set of `ReservationCheck` records — one per checkpoint defined in the listing's config. Staff work through an ordered queue of open checks.

**Check Dashboard flow:**
1. Open checks are loaded, grouped into status buckets.
2. Staff opens a check → sees full context: guest info, reservation, timeline, conversation history, instructions.
3. An AI agent pre-populates a suggested action (`AgentSuggestion`) with a recommended message and channel.
4. Staff approves or overrides → action executes (send WhatsApp, send Hostaway message, mark done, snooze, etc.).
5. Check is closed; the system moves to the next one.

### Cleaner Scheduling

- Staff create `CleaningAssignment` records mapping a cleaner (`Contact`) to a listing on a specific date.
- Assignments are approved, then published — triggering WhatsApp messages via WATI to the cleaner.
- The `CopyAssignments` view lets staff clone a prior period's schedule.
- Cleaners access their schedule at `/contacts/[contacthash]/cleanings`.

### Property Maintenance Checks

Separate from reservation checks, properties have `ListingCheck` records for recurring maintenance tasks. Issues (`ListingIssue`) are tracked, resolved, and tied back to checks.

### Flixchat (Staff Messaging)

An internal messaging hub that aggregates WhatsApp conversations with cleaners and staff. Polls for new messages every 10 seconds. Supports composing messages with a "Felix" AI assistant.

### AI Prompt Management

The `/prompts` page provides a UI for staff to view and edit the prompt templates fed to the AI agent that generates `AgentSuggestion` records. A `Snapshot` view shows historical suggestion data.

### Multi-Channel Messaging

| Channel | Integration | Use case |
|---|---|---|
| Hostaway / OTA | Hostaway API | Guest messages via Airbnb |
| WhatsApp | WATI Business API | Guest & cleaner direct messages |
| Phone | Manual log | Voice calls |

WATI message composition uses template-based and session-based messages (`WatiTemplateCompose`, `WatiSessionMessageComposer`).

---

## 6. Internationalization

Supported languages: **English**, **Spanish**, **Greek**

- Implemented via `next-intl` with message files in `/messages/`.
- Language detected from `Accept-Language` header; overridable via cookie.
- Used primarily in the cleaner-facing portal pages.

---

## 7. Data Fetching Pattern

All API calls use a consistent wrapper built on `ahooks/useRequest`:

```typescript
// GET with auto-refresh
const { data } = useGetData<Partner[]>('/v1/partners')

// GET with query params
const { data } = useFindData<ReservationCheck[]>('/v1/checks/reservations/open', { partnerId })

// POST mutation
const { run } = usePostData('/v1/assignments/cleaning/edit')
```

The wrappers handle loading state, error state, and optional polling intervals — most views poll every 10–30 seconds to stay fresh.

---

## 8. Configuration & Environment

### Environment Variables (`flixbnblobby/.env`)

```
CONTROL_ROOM_BACKEND_URL=http://localhost:3001
NEXT_PUBLIC_CONTROL_ROOM_BACKEND_URL=http://localhost:3001
WATI_TOKEN=<WATI JWT token>
```

### Firebase (`flixbnb`)

Firebase project `flixbnb-fbdf1` is used for Analytics and Hosting. Config lives in `App.tsx`.

### Next.js API Routes (Proxies)

`/api/proxy-image` and `/api/proxy-safari-video` proxy external media to fix CDN and Safari video compatibility issues.

---

## 9. Deployment

### `flixbnb` (Marketing)

```
npm run build       # Vite → /dist
firebase deploy     # Upload dist to Firebase Hosting
```

### `flixbnblobby` (Control Platform)

```
npm run build       # Next.js build
# Deploy to Vercel or any Node.js host
```

---

## 10. System Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Internet                             │
└──────────────┬──────────────────────────────┬──────────────┘
               │                              │
       ┌───────▼──────┐              ┌────────▼────────┐
       │  flixbnb     │              │  flixbnblobby   │
       │  (Marketing) │              │  (Control App)  │
       │  Firebase    │              │  Next.js 15     │
       │  Hosting     │              │  (Vercel/Node)  │
       └──────────────┘              └────────┬────────┘
                                              │
                              ┌───────────────┼───────────────┐
                              │               │               │
                     ┌────────▼──────┐ ┌─────▼──────┐ ┌─────▼──────┐
                     │  Flixbnb      │ │  Hostaway  │ │   WATI     │
                     │  Backend API  │ │  PMS API   │ │  WhatsApp  │
                     │  :3001        │ │            │ │  API       │
                     └────────┬──────┘ └────────────┘ └────────────┘
                              │
                     ┌────────▼──────┐
                     │  Neon         │
                     │  Postgres     │
                     └───────────────┘
```

---

## 11. Partners

Four property management companies use the platform:

| Partner | ID | Notes |
|---|---|---|
| Ran | 1 | Can create new listings |
| Click & Keys | 2 | |
| WaveStay | 3 | |
| UrbanKey | 4 | |

---

*Document generated from source code exploration of `flixbnb` and `flixbnblobby`.*
