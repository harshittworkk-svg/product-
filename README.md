# DAX — multi-business AI assistant

DAX is a configuration-driven, multi-tenant AI assistant starter. It uses one shared AI core for all businesses; business differences live in tenant configuration, approved knowledge, staff routing and integrations.

## Included

- NestJS API with tenant-scoped Prisma/PostgreSQL data model.
- OpenAI chat integration with approved-knowledge-only context and token/model usage records.
- Explicit approval workflow for bookings, discounts, rewards, payments, and commitment-making messages.
- Sequential staff escalation (one level at a time).
- Website chat, official Meta WhatsApp Cloud API and Telegram webhook adapters.
- Voice transcription endpoint (text response only).
- Razorpay payment-link creation and signed webhook verification.
- Audit logs, tenant IDs on business records, and soft-delete posture.
- Docker Compose for PostgreSQL/pgvector and Redis; Vitest unit tests.

## Quick start

Requirements: Node.js 20+, npm 10+, Docker.

```bash
cp .env.example .env
# Set JWT_SECRET, ENCRYPTION_KEY and integration API keys in .env
docker compose up -d
npm install
npm run db:generate
npm run db:migrate
npm run db:seed
npm run dev
```

API runs at `http://localhost:4000`. Health: `GET /health`.

The seed creates a demo tenant and admin user. For safety, the seed password is printed only by the seed script; change it immediately and never use seeded credentials in production.

## Main API

All business APIs require `Authorization: Bearer <JWT>` except health, login, public website chat, and provider webhooks. The JWT includes `tenantId`, `sub`, and role. Tenant identity is derived from the authenticated token, never accepted from a client-supplied tenant header.

| Method | Route | Purpose |
|---|---|---|
| POST | `/auth/login` | Login with business slug and receive tenant-scoped JWT |
| GET/POST | `/business/config` | Read/update tenant configuration |
| GET/POST | `/knowledge` | List/add approved knowledge |
| POST | `/chat` | Customer conversation turn |
| GET | `/conversations` | List tenant conversations |
| POST | `/actions` | Create a proposed action (never executes it) |
| POST | `/actions/:id/approve` | Human approval and execution |
| POST | `/actions/:id/reject` | Reject proposal |
| POST | `/payments/link` | Propose payment link; approval required |
| POST | `/payments/:id/approve` | Approve and create Razorpay link |
| POST | `/payments/webhook` | Verify Razorpay webhook |
| POST | `/escalations/:conversationId/next` | Route to next configured staff level |
| POST | `/voice/transcribe` | Transcribe audio; response remains text |
| POST | `/webhooks/whatsapp` | Meta WhatsApp Cloud API webhook |
| POST | `/webhooks/telegram` | Telegram bot webhook |

## Safety model

1. **No action without confirmation:** model output is treated as a suggestion. Side effects happen only in approval handlers.
2. **Knowledge grounding:** only approved tenant knowledge is passed to the model. If no useful knowledge exists, DAX says it does not know and offers human escalation.
3. **No invented commitments:** model is instructed not to commit to prices, schedules, policies, bookings or discounts.
4. **Sequential escalation:** each escalation call advances exactly one configured staff level.
5. **Channel parity:** adapters call the same `ChatService`; channel metadata is transport-only.
6. **Official channels:** WhatsApp uses Meta Cloud API; no QR/session scraping integrations are included.
7. **Voice in, text out:** transcription is supported; speech synthesis is intentionally absent.
8. **Data integrity:** business records are soft-deleted where applicable; there is no destructive customer-data endpoint. Tenant-scoped database queries and composite relations enforce separation.
9. **Cost visibility:** each model call records model, input/output tokens and estimated cost. Configure pricing in `packages/ai-core/src/pricing.ts`.
10. **Single brand:** customer-facing responses identify as Dax; business name and approved details may be included as context, not a separate AI identity.

## Payment gateway

Razorpay payment links are created only after an authenticated staff member approves the proposal. Configure `RAZORPAY_KEY_ID`, `RAZORPAY_KEY_SECRET`, and `RAZORPAY_WEBHOOK_SECRET`. Webhook signatures are verified using the raw request body. Do not mark a payment paid based only on a browser redirect.

The current implementation creates payment links and records verified payment events. Refund execution is deliberately not wired to an automatic path; refunds must remain a human-approved operational process until your refund policy, idempotency and reconciliation requirements are finalized.

## Production checklist

This is a complete runnable MVP foundation, not a security certification. Before production:
- Add managed secrets/KMS, TLS, WAF, rate limiting and provider IP/signature controls.
- Use a production identity provider, MFA, password reset and account invitation flows.
- Add a durable queue/outbox for channel delivery, retries and dead-letter handling.
- Configure database backups, point-in-time recovery, retention and restore drills.
- Complete Razorpay live-mode KYC and webhook configuration; test duplicate/out-of-order webhooks.
- Add provider-specific consent templates and opt-out handling for WhatsApp/Telegram.
- Review privacy notices, data residency, retention and applicable laws.
- Add per-tenant quotas, model spend caps and alerting.
- Add full integration/e2e tests and security review.

## Architecture

```text
Channels (Web / Meta WhatsApp / Telegram / Voice)
                    |
              Channel adapters
                    |
                ChatService
                    |
       Tenant config + approved knowledge
                    |
            Shared DAX AI core
                    |
       Proposed response / proposed action
                    |
             Human approval gate
                    |
   Booking / Discount / Reward / Payment / Message
                    |
          Audit log + usage ledger
```

## Development notes

- Business-specific behavior belongs in `BusinessConfig`, `KnowledgeDocument`, staff escalation configuration and approved templates. Do not add industry-specific branches.
- All write paths must derive `tenantId` from authenticated identity or a verified channel-to-tenant mapping.
- Keep secrets out of source control. Never log payment secrets, tokens, audio payloads or full card data.
