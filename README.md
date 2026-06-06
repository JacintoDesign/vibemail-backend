# VibeMail Backend

A data liberation sync engine that extracts Gmail data via OAuth 2.0 into Supabase and exposes a REST API. Messages are pushed in real time via Google Cloud Pub/Sub webhooks (no polling), normalized to a canonical `Message` shape, and persisted in Supabase PostgreSQL. The API is deployed as Vercel Serverless Functions.

## Stack

| Layer | Choice |
|---|---|
| Runtime | Node.js 24 |
| Language | TypeScript (strict mode) |
| Gmail integration | `googleapis` — OAuth 2.0 + Gmail API |
| Database | Supabase (PostgreSQL via `@supabase/supabase-js` v2) |
| Testing | Jest + Supertest + `ts-jest` |
| Deployment | Vercel Serverless Functions |

## API Endpoints

All endpoints are under `/api/v1` and require `Authorization: Bearer <jwt>` except the OAuth callback.

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/auth/google/callback` | OAuth 2.0 callback — exchanges code for tokens, issues JWT |
| `GET` | `/api/v1/messages` | List messages (cursor-based pagination, optional `labelId` filter) |
| `POST` | `/api/v1/messages` | Send a new email or threaded reply |
| `PATCH` | `/api/v1/messages/:id` | Mark read/unread, star/unstar, archive, or trash |
| `GET` | `/api/v1/messages/search` | Search messages by subject, from, to, or snippet |
| `GET` | `/api/v1/threads/:threadId` | Get all messages in a thread (oldest-first) |
| `POST` | `/api/v1/drafts` | Create a Gmail draft |
| `PATCH` | `/api/v1/drafts/:id` | Update a draft's to/subject/body |
| `DELETE` | `/api/v1/drafts/:id` | Delete a draft (Gmail + Supabase atomically) |
| `POST` | `/api/v1/drafts/:id/send` | Send a draft — transitions status to `sent`, clears `draftId` |

The Pub/Sub webhook receiver lives at `/webhook/gmail` (not under `/api/v1`) and validates `GOOGLE_PUBSUB_VERIFICATION_TOKEN`.

## Getting Started

### Prerequisites

- Node.js 24
- A Google Cloud project with the Gmail API and Pub/Sub enabled
- A Supabase project
- Vercel CLI (`npm i -g vercel`)

### Setup

```bash
git clone https://github.com/JacintoDesign/vibemail-backend.git
cd vibemail-backend
npm install
cp .env.example .env
```

Fill in `.env`:

| Variable | Description |
|---|---|
| `GOOGLE_CLIENT_ID` | Google OAuth 2.0 client ID |
| `GOOGLE_CLIENT_SECRET` | Google OAuth 2.0 client secret |
| `GOOGLE_REDIRECT_URI` | OAuth callback URL (e.g. `http://localhost:3000/api/v1/auth/google/callback`) |
| `GOOGLE_PUBSUB_TOPIC` | Full Pub/Sub topic name for Gmail push notifications |
| `GOOGLE_PUBSUB_VERIFICATION_TOKEN` | Shared secret to validate inbound Pub/Sub payloads |
| `SUPABASE_URL` | Supabase project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | Service-role key (bypasses RLS — server-only) |
| `JWT_SECRET` | Secret used to sign/verify JWTs (HS256) |
| `ENCRYPTION_KEY` | 64-char hex string (32 bytes) — AES-256-GCM key for OAuth tokens at rest |
| `FRONTEND_URL` | OAuth callback redirects here after issuing the JWT |

### Local Development

```bash
vercel dev        # Start local preview on port 3000
npm test          # Run Jest suite
npx tsc --noEmit  # Type-check without emitting
```

## Architecture

```
api/          Vercel Function entry points (thin handlers — no business logic)
src/          Business logic
  auth/       OAuth 2.0 flow, token exchange, token persistence
  gmail/      Provider abstraction interface + Gmail implementation
  sync/       Initial sync and history.list delta processing
  webhook/    Pub/Sub push receiver
  send/       Message composition and delivery
  drafts/     Draft create/update/delete/send
  db/         Supabase client and query helpers
  middleware/ JWT verification
migrations/   Supabase migration SQL
tests/        Jest integration and unit tests
```

Messages are ingested exclusively via Pub/Sub push webhooks — the receiver decodes the `historyId`, calls `history.list` for the delta, and upserts changed messages into Supabase. There is no polling.

The `status` field on every `Message` is derived server-side from `labelIds` at write time using this priority order:

```
'draft'    → labelIds includes DRAFT
'sent'     → labelIds includes SENT (and not DRAFT)
'trash'    → labelIds includes TRASH
'archived' → not INBOX, SENT, DRAFT, or TRASH
'inbox'    → default
```

## Error Shape

Every non-2xx response returns:

```json
{
  "error": {
    "code": "SCREAMING_SNAKE_CASE",
    "message": "Human-readable description",
    "details": {}
  }
}
```

## Testing

```bash
npm test                                    # Full suite
npx jest --testPathPattern=drafts           # Single file
```

The schema migration must be applied to the test database before integration tests can pass (see `CONTRACT.md §2` for the two-session sequencing rule).

## Deployment

```bash
vercel --prod
```

Set all environment variables in the Vercel project dashboard before deploying. The `SUPABASE_SERVICE_ROLE_KEY` and `ENCRYPTION_KEY` are sensitive — never commit them.
