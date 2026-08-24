# CFDI Express API — Technical Architecture & Build Plan

**Version:** 0.1 (draft for review) · **Date:** 2026-07-28 · **Author:** Acromático Development
**Canonical product spec:** [`rfp-cfdi-api.md` v1.5](../../acromatico-cfdi-express/docs/rfp-cfdi-api.md) — this document is the technical counterpart: how we build it, with what, and in which order.

---

## 1. Executive summary

CFDI Express API is a multi-tenant REST API for CFDI 4.0 invoicing (income invoices PUE/PPD, payment complements REP, SAT cancellations), stamped through **SW Sapien**, billed 100% by usage through **Stripe prepaid credits**. It must comfortably absorb bursts of **100 concurrent invoice requests**, deliver **signed webhooks**, and ship with a **client dashboard** (credits, usage, API keys, merchants, webhooks) plus a minimal internal admin.

**Scope guardrail — no buyer communication:** the platform issues fiscal documents and exposes them (signed URLs, webhook events). It **never communicates with the buyer** — no email, no WhatsApp, no delivery channel of any kind. Delivering the CFDI to the end customer is the API consumer's job: they fetch the zip/PDF/XML from the signed URLs (or from the `invoice.stamped` webhook payload) and send it from their own systems. Consequently, the invoice payload carries **fiscal receiver data only** — no buyer contact channels are collected or stored. The one exception is **operational email to the API client** (credit alerts, CSD expiry, webhook endpoint health, dashboard auth), which stays.

**Key decisions at a glance:**

| Area | Decision | Status |
|---|---|---|
| HTTP framework | **Hono** on Node 22 + `@hono/zod-openapi` (OpenAPI 3.1 generated from code) | Endorses proposal |
| Queues | **BullMQ** + Redis 7 — also used as the load-regulation mechanism in front of the PAC | Endorses proposal |
| Stamping path | **Queue-mediated synchronous stamping**: request → stamp queue → await job (BullMQ `QueueEvents`) → 201; graceful degradation to `202 Accepted` under extreme load | New — core of the 100-concurrent design |
| PDF | **Gotenberg** (existing instance) — stateless HTML→PDF over HTTP, called from queue workers, never from the request path | Per team decision; replaces in-process Puppeteer |
| Webhooks | **Built in-house**: transactional outbox (Postgres) + BullMQ delivery workers + HMAC-SHA256 Stripe-style signatures. Svix evaluated as managed alternative — rejected for v1 | New recommendation |
| Database | **PostgreSQL 16 + Prisma** (per RFP), tenant-scoped via Prisma extension | Per RFP |
| File storage | **Cloudflare R2** (S3-compatible, zero egress) with on-demand presigned URLs | Recommendation (RFP said "PocketBase or S3-compatible") |
| Credits ledger | Postgres ledger + **atomic conditional decrement** (reserve → commit/refund) — race-safe under concurrency | Design detail |
| Idempotency | **Persistent Postgres idempotency keys** (Stripe-style, response replay) + short Redis locks | Design detail |
| Dashboard | React Router v7 + Tailwind/shadcn + better-auth, in the same monorepo | New recommendation |
| Observability | pino + Sentry + Prometheus metrics; SW stamp-balance monitor with alerts | Per RFP + detail |
| Deployment | **Self-hosted VPS + Docker + Coolify (Traefik)** — all services as containers on the VPS; docker-compose for local dev | Team decision |

Total estimate (unchanged from RFP §7): **~14–18.5 weeks**, preceded by a **2–3 day spike** (RFP §10) that now also includes a load/latency measurement against the SW sandbox.

---

## 2. Architecture at a glance

```
                        ┌────────────────────────────────────────────────────────┐
                        │                  CFDI Express API                       │
                        │                                                         │
 Client apps ──HTTPS──▶ │  apps/api (Hono)                                        │
 (Authorization:        │   ├─ auth (API keys) · rate limit · idempotency gate    │
  Bearer sk_live_…)     │   ├─ credit admission check (balance + monthly limit)   │
                        │   └─ routes /v1/*                                       │
                        │        │ enqueue + await (QueueEvents, ~20 s)           │
                        │        ▼                                                │
                        │  apps/worker (BullMQ)                                   │
                        │   ├─ stamp        → SW Sapien issue/cancel (throttled)  │
                        │   ├─ postprocess  → PDF (Gotenberg) · zip · R2          │
                        │   ├─ webhooks     → signed HTTP delivery + retries      │
                        │   ├─ cancel-poll  → SAT cancellation status polling     │
                        │   └─ scheduler    → rep_due · csd.expiring · balances   │
                        │                                                         │
                        │  apps/web (React Router dashboard + internal admin)     │
                        └──────┬──────────┬───────────┬───────────┬──────────────┘
                               │          │           │           │
                        PostgreSQL     Redis 7    Cloudflare R2  Gotenberg
                        (Prisma)     (BullMQ +   (XML/PDF/zip/  (existing
                                      rate limit  acuses,       instance)
                                      + locks)    presigned)
```

**Deployment:** the three app services (api, worker, web) are **Docker containers on a VPS managed by Coolify**, with **Traefik** terminating TLS and routing the public hostnames (`api.cfdi.express` for the API, `api.dash.cfdi.express` for the dashboard). Backing services (PostgreSQL, Redis, Gotenberg) are **deployed independently** — Gotenberg already exists, Redis/Postgres may be shared with other projects — and are reached through private endpoints configured via env vars (`DATABASE_URL`, `REDIS_URL`, `GOTENBERG_URL`). Cloudflare R2 is the only external infrastructure service (§14).

External services: **SW Sapien** (stamp/CSD/cancel/balance — production + `services.test.sw.com.mx` sandbox with the team's test token), **Stripe** (Checkout, off-session recharges, customer receipts), **Mailgun** (**operational account notifications only** — credit alerts, CSD expiry, webhook endpoint health, dashboard auth; reused from the Shopify app). No buyer-facing email: CFDI delivery is the API consumer's responsibility (§1 scope guardrail).

**Why this shape:** the request path only does cheap, fast work (auth, validation, DB). Everything expensive or unreliable — PAC calls, PDF rendering, webhooks — runs behind queues with explicit concurrency, retries and backpressure. That is what makes a 100-request burst a non-event.

---

## 3. Technology decisions

### 3.1 HTTP framework — Hono (endorsed)

- `@hono/node-server` on Node 22 LTS, TypeScript strict.
- **`@hono/zod-openapi`**: routes are declared with zod schemas → request/response validation **and** the OpenAPI 3.1 document come from the same source. The RFP requires an OpenAPI 3 spec (§5); this makes it impossible for docs and implementation to drift.
- Docs UI via `@scalar/hono-api-reference` served at `/docs`; Postman collection generated from the spec in CI.
- Middleware we need exists and is boring: CORS, body limit, request-id, pino logger.
- Runtime-agnostic: if we ever want a Cloudflare Worker in front (geo edge, WAF), Hono ports without rewrites.

*Alternatives considered:* Fastify (excellent, larger plugin ecosystem, but its OpenAPI story via `fastify-swagger` + JSON Schema is clunkier than zod-openapi); Express (no). Either would work; Hono is the best fit for spec-from-code.

### 3.2 Queues — BullMQ + Redis (endorsed)

Already proven in the Shopify app (`bullmq@5`, `ioredis`). We use it for **five** queue groups (§7/§8). Two BullMQ features are load-bearing in this design:

- `Worker` **concurrency + rate limiter** per queue → exact control of pressure toward SW, Gotenberg and client endpoints.
- **`QueueEvents` + `job.waitUntilFinished()`** → lets an HTTP request enqueue a stamp job and await its result, giving us queue-based backpressure with synchronous semantics (§6).

Redis also hosts: rate limiting (sliding window), short-lived locks (idempotency claim, recharge cooldown), and the Bottleneck limiter state (cluster mode).

### 3.3 PDF — Gotenberg (existing instance)

The team already operates a Gotenberg instance — a stateless container deployed independently of this project — use it.

- Render workers POST `multipart/form-data` to `POST /forms/chromium/convert/html` (Chromium route): `index.html` (our SAT template) + assets (logo fetched from R2, embedded fonts/CSS). PDF bytes come back in the response; worker uploads to R2.
- **No browser management in our processes** (no Puppeteer pool sizing, no Chromium memory leaks in workers). Gotenberg is stateless → scale it by raising container resources or adding a second container on the internal Docker network if rendering ever becomes the bottleneck. It is **never exposed publicly** — only `worker` reaches it over the internal network.
- Worker-side rules: 30 s timeout, retry up to 3× with backoff on 5xx/transport errors (HTML→PDF is idempotent — safe to retry, unlike stamping), alert if error rate spikes.
- QR code for the SAT verification URL is generated in-process (`qrcode` npm package → data-URI `<img>`), so the template is fully self-contained.
- Dev: Gotenberg runs as one more service in docker-compose.

*Alternatives considered:* in-worker Puppeteer pool (more moving parts, heavier workers); pdfmake (fast but painful layout for the SAT printed representation); Typst (great, new toolchain — revisit if Gotenberg underperforms).

### 3.4 Webhooks — built in-house (with Svix as the documented alternative)

The question "what do we use for webhook subscriptions" has two honest answers:

| | Build in-house (chosen) | Svix (managed) |
|---|---|---|
| Cost | ~0 (BullMQ already in stack) | Per-message pricing |
| Effort | ~1–1.5 weeks (delivery worker + retries + signing + dashboard page) | ~2 days integration |
| Control | Full (payload shapes, retry policy, replay UX) | Their model |
| Consumer UX | Good (signed, retried, replayable) | Excellent (embedded portal, SDKs) |

Recommendation: **build in-house for v1**, behind a narrow internal interface (`WebhookDispatcher`) so Svix can replace the internals later without changing the API surface. Volume at launch (hundreds of deliveries/minute peak) is trivial for a BullMQ worker. Full design in §9.

### 3.5 Storage — Cloudflare R2 (recommendation)

RFP allows "existing PocketBase or S3-compatible". For a public API product, choose **R2**:

- S3 API → presigned GET URLs generated on demand (15 min TTL) satisfy the RFP's "files as short-lived signed URLs" principle with zero custom auth code.
- **Zero egress fees** — clients download PDF/XML/zip a lot; S3 egress would become a real cost line.
- Not coupled to the Shopify app's PocketBase (which is that product's internal admin DB, not a file CDN).
- Dev/staging/prod use **separate R2 buckets** (`cfdi-express-api-dev` / `-staging` / `-prod`). No MinIO or local S3 emulation — R2's free tier (10 GB, 1M Class-A ops/month) covers dev volume, and developing against the real S3 API avoids emulation drift (presigning, CORS, multipart behavior).

### 3.6 Dashboard — React Router v7 + better-auth (recommendation)

The team already ships React Router v7 in production (the Shopify app). The client portal is a separate small app in the same monorepo:

- **React Router v7** (framework mode) + Tailwind + shadcn/ui (Polaris is Shopify's design language — wrong fit for a standalone product).
- **better-auth** for dashboard identity: email/password + magic links, Prisma adapter, session management, admin role support. Self-hosted, TypeScript-native, no per-MAU pricing (Clerk/Auth0 would also work; unnecessary cost for v1).
- Dashboard talks to the same domain services (`packages/core`) as the API — never a parallel implementation.
- Internal admin = same app, `role=admin` routes (accounts, manual credit adjustments, SW balance, webhook health).

### 3.7 PAC adapter — `PacProvider` interface (per RFP §6)

All SW contact behind one interface in `packages/pac`: `uploadCsd`, `stamp`, `cancel`, `cancelStatus`, `balance`. SW implementation first; the interface normalizes errors into our taxonomy (§11.4) so SAT/SW error codes map cleanly to `application/problem+json`. This is also where **retry reconciliation** lives (§6.4).

---

## 4. Monorepo layout (pnpm workspaces)

```
cfdi-express-api/
├── apps/
│   ├── api/            # Hono HTTP API (stateless, N replicas)
│   ├── worker/         # BullMQ workers (stamp, postprocess, webhooks, cancel-poll, scheduler)
│   └── web/            # React Router dashboard + internal admin
├── packages/
│   ├── db/             # Prisma schema, migrations, seeds (SAT catalogs), client + tenant extension
│   ├── core/           # Domain services: invoicing, tax engine (ported), credits/ledger,
│   │                   #   catalogs, idempotency, webhook outbox — framework-free
│   ├── pac/            # PacProvider interface + SW Sapien implementation + error normalization
│   ├── pdf/            # SAT HTML templates (invoice nominal/global, REP) + Gotenberg client
│   └── shared/         # Types, problem+json errors, env parsing (zod), constants
├── infra/
│   ├── docker-compose.yml        # local dev: full stack — postgres, redis, mailpit, gotenberg, api, worker, web
│   ├── docker-compose.prod.yml   # staging/prod: api, worker, web ONLY — backing services are env-var endpoints
│   ├── Dockerfile.{api,worker,web}
│   └── coolify/                  # per-environment compose/config consumed by Coolify (staging, production)
├── docs/
├── openapi/            # generated spec (CI artifact, committed)
└── tests/              # E2E + k6 load scenarios
```

No Turborepo for now — `pnpm --filter` scripts are enough at this size. Domain logic lives in `packages/core` so both `apps/api` and `apps/web` (and tests) consume the same code paths.

---

## 5. Data model (PostgreSQL + Prisma)

Sketch — field lists are representative, not exhaustive. Every tenant-owned table carries `accountId`; a Prisma client extension forces the scope on every query (defense in depth: add Postgres RLS in Phase 6 hardening).

| Model | Purpose / key fields |
|---|---|
| `Account` | Tenant. `stripeCustomerId`, `creditsBalance` (int), `monthlyCreditLimit?`, `autoRechargeEnabled/thresholdCredits/packageSize`, `autoRechargePaymentMethodId`, `status` |
| `User` | Dashboard identity (better-auth): email, credentials, `role` (`owner` \| `admin` internal) |
| `ApiKey` | `accountId`, `mode` (`test`/`live`), `prefix` (display), `keyHash` (SHA-256), `lastUsedAt`, `revokedAt` — pairs per RFP §3.1 |
| `Merchant` | `accountId`, RFC, razón social, régimen fiscal, CP, default `claveProdServ`/`claveUnidad`, `logoKey` |
| `Csd` | `merchantId`, encrypted cer/key/password (envelope: per-merchant DEK + KEK), `serialNumber`, `validFrom/To`, `pacCertificateId` |
| `Invoice` | Opaque `id` (cuid2). `accountId`, `merchantId`, `livemode`, `kind` (`nominal`/`global`), `status` (`stamping`/`stamped`/`stamp_failed`/`cancel_pending`/`cancelled`), `metodoPago` (PUE/PPD), `formaPago`, totals, `receiver` jsonb, `informacionGlobal` jsonb?, `uuid?`, `pacDocumentId?` (internal), `xmlKey/pdfKey/zipKey`, `filesStatus`, `saldoPendiente?` |
| `Payment` (REP) | First-class resource. `invoiceId`, `numParcialidad`, `monto`, `formaPago`, `fechaPago`, saldos anterior/insoluto, `uuid?`, `status`, file keys. Multi-document REPs via `PaymentDocument` join (§4.2 RFP edge case) |
| `Cancellation` | `documentType` (`invoice`/`payment`), `documentId`, `motivo` (01–04), `folioSustitucion?`, `status` (`pending`/`accepted`/`rejected`/`cancelled`), `acuseKey` |
| `CreditLedger` | Append-only. `type` (`purchase`/`consume`/`refund`/`recharge`/`adjustment`), `delta`, `balanceAfter`, `referenceType/Id` — the audit trail of every credit movement |
| `UsageMonthly` | `(accountId, month)` unique, `creditsUsed` — enforces the monthly cap |
| `IdempotencyKey` | `(accountId, key)` unique, `requestHash`, `status` (`in_flight`/`completed`/`failed`), stored response snapshot, `expiresAt` (72 h) |
| `WebhookEndpoint` | `accountId`, `url`, `secret` (`whsec_…`, shown once), `events[]`, `status`, `consecutiveFailures` |
| `WebhookDelivery` | Outbox row: `eventId` (unique, dedupe), `endpointId`, `eventType`, `payload` jsonb, `status`, `attempts`, `lastStatus`, `nextAttemptAt`, `deliveredAt` |
| `AuditLog` | Every mutating request: actor, action, resource, IP/UA, request-id |
| SAT catalogs | `SatProduct`, `SatUnit`, `SatUsoCfdi`, `SatRegimen`, `SatFormaPago` — seeds migrated from the Shopify app |

SAT-side identifiers: the fiscal UUID is **data on the resource** (`uuid`); the SW document id stays internal (`pacDocumentId`); clients only ever see our opaque `id` (RFP §3 design principles).

---

## 6. The stamping pipeline — designing for 100 concurrent requests

This is the heart of the system. Requirements: bursts of 100 simultaneous `POST /v1/invoices`; SW is a third party with finite throughput and no idempotency guarantees; a stamp costs real money (a SW timbre **and** a client credit); PDF rendering is heavy and must never block the request path.

### 6.1 Request lifecycle (synchronous happy path)

```
POST /v1/invoices  (Idempotency-Key: abc)
 1. Auth + rate limit + zod validation (SAT rules: RFC, régimen↔uso CFDI,
    informacionGlobal periodicity, PPD≠global, …)           → 401/429/400
 2. Idempotency gate (§10): replay / 409 / claim             → 201(replayed)|409
 3. Credit admission: atomic conditional decrement            → 402
    UPDATE "Account" SET "creditsBalance" = "creditsBalance" - 1
    WHERE id = $1 AND "creditsBalance" >= 1
    (+ monthly-cap check on UsageMonthly in the same tx)
 4. Insert Invoice(status='stamping') + enqueue stamp job
 5. await job.waitUntilFinished(queueEvents, timeout=20s)
       ├─ success → 6a
       ├─ business failure (SAT validation) → refund credit, 422 problem+json
       └─ timeout (>20 s, extreme load) → 202 { id, status:"stamping" }
 6a. 201 { id, status:"stamped", uuid, xmlUrl, files:{ status:"processing" } }
 6b. Worker has already chained the postprocess job (§6.3)
```

The client that needs the UUID immediately has it in ≤ a few seconds. The client that needs the files gets them from `GET /v1/invoices/{id}` (once `filesStatus:"ready"`) or — preferably — from the `invoice.stamped` webhook, which fires **after** post-processing and carries all download URLs (RFP §4.1 delivery semantics).

### 6.2 Why queue-mediated (instead of calling SW inline from the request)

| Property | Inline call + p-limit | Queue-mediated (chosen) |
|---|---|---|
| Burst absorption | 429/shed load when saturated | Queue buffers; requests wait or degrade to 202 |
| Cluster-wide PAC throttle | Per-pod limiter (approximate) | Exact: worker concurrency + Bottleneck (Redis, cluster mode) |
| Retries/backoff toward SW | Hand-rolled | BullMQ native (attempts, backoff, DLQ) |
| Post-stamp chaining | Fire-and-forget (leaky) | Job chains (stamp → postprocess → webhook) |
| Latency added | 0 | ~5–15 ms (Redis round-trip) |

BullMQ is already mandatory for post-processing, webhooks, cancellation polling and schedulers — putting stamping behind the same fabric costs one `QueueEvents` await and buys all of the above.

### 6.3 Worker chain per stamped document

`stamp` worker (concurrency 25/pod, SW-throttled) → on success: persist XML + uuid, **commit** the reserved credit (ledger `consume` entry), chain `postprocess` job → render PDF via Gotenberg → build zip (jszip, reused) → upload XML/PDF/zip to R2 → write webhook outbox rows (`invoice.stamped`) in the same DB tx as `filesStatus="ready"`. No email is sent to the buyer — the client picks the files up via the signed URLs or the webhook payload.

Each REP follows the identical chain (`payment.stamped`).

### 6.4 Stamp retries without double-stamping (critical)

SW's issue endpoint is **not idempotent**; a retry after an ambiguous failure (timeout, connection reset after the request was sent) could mint two fiscal documents. Rules in `packages/pac`:

1. Auto-retry (≤3, backoff) **only** for pre-send failures (DNS, connect-refused, 5xx before body sent).
2. On ambiguous post-send failure: **reconcile before retry** — query SW by our unique internal `folio` (we control serie/folio per merchant) to learn whether the stamp actually landed; if yes, adopt that result; only if definitively absent, retry once.
3. Never retry SAT validation errors (4xx) — they are definitive; surface as 422.
4. Every attempt logged with request-id for forensics.

### 6.5 Cancellation path (always async — RFP §3.5)

`POST /v1/invoices/{id}/cancel` (or `/v1/payments/{id}/cancel`) → validate (motivo 01–04, folio for 01, PPD-with-REPs orchestration rule) → create `Cancellation(status:"pending")` + enqueue → worker calls SW cancel → store acuse → `202 Accepted` immediately. A `cancel-poll` worker re-checks status hourly (≤72 h) because cancellations >$1,000 MXN may need receptor acceptance; transitions emit `invoice.cancelled` / `invoice.cancellation_pending`.

### 6.6 Capacity math (validate against the sandbox in the spike)

Assumptions to verify: SW stamp p95 ≈ 2–4 s; Gotenberg p95 ≈ 0.5–2 s per page.

| Component | Sizing (single VPS) | Burst of 100 concurrent stamps |
|---|---|---|
| api containers | 2 replicas behind Traefik, stateless | Hold 100 keep-alive requests trivially (I/O wait only) |
| stamp workers | 1–2 containers, concurrency 25 each, Bottleneck cap at SW's real limit | 4 waves × ~3 s ≈ **12–16 s to drain**; 202 fallback only if >20 s |
| Gotenberg | existing instance (independent deployment) | ~100 PDFs drain in <1 min behind the stamps (render is off critical path) |
| PostgreSQL | independent service; pool 10/app | ~6–8 rows written per stamp — trivial |
| Redis | independent service (AOF recommended) | ~50 BullMQ ops/job — trivial |

Sustained headroom at concurrency 25 ≈ 6–12 stamps/s ≈ **20–40 M timbres/month theoretical** — the binding constraint will be the SW contract and client credit balances, not our infra. The throttle ceiling is a config value we raise once SW confirms limits. VPS scaling is vertical first (resize), then service split (§14).

---

## 7. Queue topology

| Queue | Producer | Worker concurrency | Retry policy | Notes |
|---|---|---|---|---|
| `stamp` | api | 25/pod + global Bottleneck | §6.4 rules; DLQ after final failure → `stamp_failed` + refund + webhook | Job data: `invoiceId` only (payload from DB) |
| `postprocess` | stamp worker (chain) | 10/pod | 5 attempts, exp. backoff → DLQ + ops alert (invoice stays `stamped`, files recoverable) | PDF → zip → R2 → outbox |
| `webhooks` | outbox dispatcher | 50/pod | 8 attempts: 0, 1m, 5m, 30m, 2h, 6h, 12h, 24h → `failed` | §9 |
| `cancel-poll` | cancel worker / scheduler | 5/pod | Hourly re-check ≤72 h | SAT acceptance windows |
| `scheduler` | repeatable jobs | 1 (singleton) | standard | `payment.rep_due` daily scan, `csd.expiring` (30/15/7/1 d), SW stamp-balance hourly, stranded-outbox sweeper (30 s), Stripe recharge retries |
| `stripe` | api (Stripe webhook intake) | 5/pod | 5 attempts | Off-session recharge side effects |

Operational rules: every queue has a DLQ with alerting; jobs carry only IDs (state lives in Postgres); workers are idempotent consumers (job may run twice → effects guarded by DB state transitions).

---

## 8. Webhook subsystem (in-house build)

### 8.1 Model

- Client manages endpoints via dashboard (and `POST /v1/webhook_endpoints` if we expose it): `url`, subscribed `events[]`, gets back a `whsec_…` secret **shown once** (rotation keeps two secrets valid for 24 h).
- Events (RFP §3.6): `invoice.stamped`, `invoice.stamp_failed`, `invoice.cancelled`, `invoice.cancellation_pending`, `payment.stamped`, `payment.rep_due`, `csd.expiring`, `credits.low`, `credits.depleted`, `credits.limit_reached`, `credits.recharge_failed`.

### 8.2 Transactional outbox (delivery guarantee)

Domain changes and their `WebhookDelivery` rows commit in the **same Postgres transaction** → a committed state change can never lose its notification. A dispatcher enqueues due rows to the `webhooks` queue; a 30-second sweeper re-enqueues anything stranded (covers Redis enqueue failures after commit). `eventId` is unique → replay/dedupe is trivial.

### 8.3 Delivery & signing (Stripe-style)

```
POST {endpoint.url}
CFDI-Event: invoice.stamped
CFDI-Delivery: del_01J…        (idempotent consumer key)
CFDI-Signature: t=1785400000,v1=<hex HMAC-SHA256(secret, "{t}.{raw body}")>
Content-Type: application/json

{ "id": "evt_01J…", "type": "invoice.stamped", "livemode": true,
  "created": 1785400000, "data": { "object": { …invoice… } } }
```

- 10 s timeout; any non-2xx = failure → retry schedule from §7.
- **Auto-disable** after 100 consecutive failures (email + `credits`-style notice; re-enable from dashboard).
- Docs include verification snippets (Node/PHP/Python) and consumer guidance: dedupe on `CFDI-Delivery`, tolerate out-of-order delivery (check `created`), respond fast and process async.
- Dashboard: delivery log per endpoint (status, response code, latency), **manual replay** button, "send test event".

---

## 9. Idempotency (API-level, persistent)

Stronger than the Shopify app's TTL gate — a public API needs durable semantics:

- `Idempotency-Key` header on all POSTs (required on `/v1/invoices`, `/payments`, cancel actions).
- `(accountId, key)` row claimed by atomic insert before any side effect:
  - key exists + `status:completed` + same `requestHash` → **replay** the stored response (`Idempotency-Replayed: true`).
  - key exists + different `requestHash` → `422` (`idempotency_key_reuse`).
  - key exists + `in_flight` → `409` (`request_in_flight`).
- Rows expire after 72 h. A short Redis lock (`SET NX PX`) around the stamp enqueue additionally collapses true simultaneous duplicates.
- Combined with §6.4 reconciliation, double-stamping is defended at three layers: client key → DB claim → PAC-level folio reconciliation.

---

## 10. Credits, limits & Stripe

- **Reserve/commit/refund**: admission decrement (§6.1 step 3) = reserve; stamp success = commit (`CreditLedger consume`); failure = refund (`+1`, ledger `refund`). Failed stamps never consume credits (RFP §3.7) and balances can never go negative — the conditional `UPDATE … WHERE balance >= 1` is atomic under concurrency.
- **Monthly cap**: `UsageMonthly` incremented in the same transaction; exceeding the cap → `402` + `credits.limit_reached`.
- **Purchase**: packages (50/200/1,000…) as Stripe one-time Prices → Stripe Checkout → webhook `checkout.session.completed` credits the ledger (idempotent on Stripe event id).
- **Auto-recharge**: balance after commit `< threshold` → enqueue recharge (Redis cooldown lock, one in flight) → off-session PaymentIntent on the saved PaymentMethod (SetupIntent at activation) → success: ledger `purchase`; `authentication_required`/failure: `credits.recharge_failed` + email + retry next scheduler pass; balance 0 → stamping blocked (`402` + `credits.depleted`).
- **Threshold alerts**: `credits.low` crossing is detected at commit time (one-shot per crossing).
- Stripe webhook intake: `/webhooks/stripe` (unversioned), signature-verified, event-id deduped, enqueued to the `stripe` queue for side effects.
- Receipts/invoices for credit purchases: surfaced in the dashboard from Stripe (hosted invoice URLs) — no PDF work on our side for v1.

---

## 11. Security

1. **API keys**: `sk_live_`/`sk_test_` + 32 random bytes; stored SHA-256; lookup by prefix + constant-time hash compare; shown once; rotation = create new → revoke old.
2. **CSD at rest**: envelope encryption — per-merchant AES-256-GCM DEK wrapped by a KEK in KMS (or age-encrypted master key in secrets manager). Passwords never plaintext, never logged. Local validation before upload: cer/key correspondence, validity window, RFC extracted from cert, CSD-not-FIEL check (key-usage bits — confirm heuristic in spike).
3. **Tenant isolation**: `accountId` scoping enforced by a Prisma client extension (no raw cross-tenant queries); Postgres RLS as Phase 6 hardening; every query in tests asserts isolation.
4. **Rate limiting**: per-key sliding window in Redis (default 300 rpm, configurable per account); `429` + `Retry-After` + problem+json. Stamp admission additionally bounded by queue depth (§6).
5. **Audit**: append-only `AuditLog` for all mutations (actor, IP, UA, request-id); 90-day hot retention.
6. **Webhooks**: per-endpoint HMAC secrets, timestamp tolerance (5 min) against replay, secret rotation.
7. **Transport**: TLS everywhere, HSTS, 1 MB body limit, zod-validated inputs only.
8. **Stripe**: webhook signature verification; no card data ever touches our servers (Checkout/Elements hosted fields).
9. **Secrets**: Doppler/1Password Connect (align with team's current setup) injected as env; nothing in the repo.

### 11.4 Error taxonomy (`application/problem+json`, RFC 7807)

`type` URLs → docs pages; stable machine `code`s: `validation_error` (with SAT field pointers), `sat_rejected`, `insufficient_credits`, `monthly_limit_reached`, `pac_unavailable`, `idempotency_key_reuse`, `request_in_flight`, `rate_limited`. SW/SAT errors normalized once in `packages/pac` — routes never parse vendor payloads.

---

## 12. API surface (v1)

Base: `https://api.cfdi.express` — environment by key (`sk_test`/`sk_live`), `livemode` flag on every object (Stripe pattern; RFP §3). All lists: cursor pagination (`?limit=&starting_after=`).

| Resource | Endpoints |
|---|---|
| Invoices | `POST /v1/invoices` · `GET /v1/invoices` (filters: merchant, dates, status, uuid, método) · `GET /v1/invoices/{id}` (incl. `cancellation`, `saldoPendiente`, signed file URLs) · `POST /v1/invoices/{id}/cancel` |
| Payments (REP) | `POST /v1/invoices/{id}/payments` · `GET /v1/payments/{id}` · `POST /v1/payments/{id}/cancel` |
| Merchants | `GET/POST /v1/merchants` · `GET/PATCH/DELETE /v1/merchants/{id}` · `POST/PUT /v1/merchants/{id}/csd` · `GET /v1/merchants/{id}/csd` (metadata/vigencia) |
| Catalogs | `GET /v1/catalogs/{products,units,usos-cfdi,regimenes,formas-pago}?keyword=` (local SAT data) |
| Webhooks | `GET/POST /v1/webhook_endpoints` · `GET/PATCH/DELETE /v1/webhook_endpoints/{id}` · `GET /v1/webhook_endpoints/{id}/deliveries` |
| Billing | `GET /v1/balance` · `GET /v1/usage?month=` (dashboard consumes the same services) |
| Meta | `GET /health` · `GET /ready` · `GET /docs` (Scalar) · `GET /openapi.json` |

Response contract detail for `POST /v1/invoices`: `201` → `status:"stamped"`, `uuid`, `xmlUrl`, `files:{status:"processing"}`; `202` → `status:"stamping"` (extreme load only); `GET` later → `files:{status:"ready", pdfUrl, xmlUrl, zipUrl}`.

---

## 13. Dashboard & internal admin (apps/web)

**Client portal (v1 scope, RFP §3.1):**
- Overview: credit balance, current-period consumption vs monthly cap, recent invoices.
- Credits: buy packages (Stripe Checkout redirect), auto-recharge config (threshold + package + card via SetupIntent/Elements), monthly cap setting, ledger history, Stripe receipts.
- API keys: create/revoke/rotate, live vs test, last-used.
- Merchants: CRUD + CSD upload wizard (validation feedback, vigencia display, expiry warnings).
- Invoices/REPs: searchable list, detail with downloads, cancel with motivo flow.
- Webhooks: endpoints CRUD, delivery log with payload inspector, replay, test event.

**Internal admin (same app, `role=admin`):** accounts list/impersonate-read-only, manual credit adjustments (ledger `adjustment` + reason, audit-logged), SW stamp-balance gauge + alert config, webhook delivery health across tenants, DLQ inspection, kill switches (disable stamping per account).

---

## 14. Environments & deployment

**All services are self-hosted on a VPS with Docker + Coolify, fronted by Traefik** (team decision). Coolify manages deployments from the GitHub repo; Traefik terminates TLS (Let's Encrypt) and routes public hostnames to containers; everything else stays on the internal Docker network.

- **Dev**: `docker compose up` (`infra/docker-compose.yml`) starts the **full stack**: Postgres 16, Redis 7, Mailpit, Gotenberg, api, worker, web — plus the dev R2 bucket for files (no local S3 emulation, §3.5). Seed script loads SAT catalogs + a test account/merchant/CSD.
- **Test mode vs live mode** (RFP §5): one deployment serves both; `sk_test_*` → SW sandbox (`services.test.sw.com.mx`, team test token) + Stripe test keys + `livemode:false` data; `sk_live_*` → production SW + Stripe. Data separated by the `livemode` discriminator, never by environment.
- **Staging**: a second deployment of the **same prod compose** (app containers only) in Coolify, pointing at staging backing endpoints/databases + SW sandbox + Stripe test; used by E2E and k6.
- **Production — app containers** (`infra/docker-compose.prod.yml`, deployed as a Coolify project; the only services this repo ships in prod):

  | Service | Exposure | Notes |
  |---|---|---|
  | `api` ×2 replicas | `api.cfdi.express` via Traefik (TLS) | Stateless; replicas behind Traefik |
  | `web` ×1 | `api.dash.cfdi.express` via Traefik (TLS) | Dashboard |
  | `worker` ×1–2 | none (internal) | All queue groups; scale by container count |

- **Backing services — deployed independently** (NOT in this repo's prod compose; each is provisioned/managed on its own and injected as an env-var endpoint):

  | Service | Env var | Notes |
  |---|---|---|
  | PostgreSQL 16 | `DATABASE_URL` | Independently deployed; nightly `pg_dump` → R2 |
  | Redis 7 | `REDIS_URL` | Dedicated or shared instance; AOF persistence recommended |
  | Gotenberg | `GOTENBERG_URL` | Already deployed (existing instance) |
  | Cloudflare R2 | `S3_ENDPOINT` / `S3_BUCKET` / keys | External; per-environment buckets (§3.5) |

  Dev composes those same services as local containers; staging/prod compose only the app containers. Same images, same env-var contract — the only difference between the two compose files is whether the backing services run alongside or are external endpoints.

- **CI/CD (GitHub Actions)**: typecheck → lint → unit → build images → push to GHCR → Coolify deploy webhook (staging auto-deploys from `main`; production promotes manually by tag). Coolify can alternatively build from the repo Dockerfiles itself — pick one build path in Phase 0 and stick to it.
- **Migrations**: `prisma migrate deploy` as a pre-deploy one-off task (Coolify one-time command against `DATABASE_URL`); backwards-compatible migrations only (expand/contract).
- **Backups**: nightly `pg_dump` to an R2 bucket with retention + provider VPS snapshots. Redis data is rebuildable (queues re-drive from Postgres state); Postgres is the source of truth.
- **Secrets**: Coolify per-service env vars (`DATABASE_URL`, `REDIS_URL`, `GOTENBERG_URL`, `S3_*`, SW production token, **SW test token for `services.test.sw.com.mx`**, Stripe live/test keys, KEK/master key, Mailgun, Sentry DSN) — never in the repo.
- **Scaling path**: start with a ~4 vCPU / 8 GB VPS for the app containers; backing services scale on their own deployments. The 100-concurrent burst fits comfortably in the starting size (§6.6).

---

## 15. Observability & ops

- **Logs**: pino JSON everywhere, `request-id` propagated API → queue → PAC call.
- **Errors/APM**: Sentry in api/worker/web (team already operates it).
- **Metrics** (`/metrics`, prom-client): stamp latency p50/p95/p99, SW error rate, queue depth + oldest-job age per queue, webhook delivery success rate, credit operations/min, Gotenberg latency/error rate.
- **Alerts**: SW stamp balance below threshold (hourly Balance V2 poll — ops buys timbres prepay); queue depth/age; webhook failure spike; stamp failure rate; 5xx rate; DB pool saturation; Gotenberg down.
- **Health**: `/health` (liveness), `/ready` (DB+Redis shallow checks) — wired to Coolify/Traefik healthchecks and external uptime monitoring; public status page later.
- **Runbooks** (written in Phase 6): SW outage mode (queues back up, API keeps accepting → 202s), Redis outage (fail-closed on stamping, documented), double-stamp incident, DLQ replay procedure.

---

## 16. Testing strategy

- **Unit (Vitest)**: the ported tax engine gets a golden-master suite from production Shopify-app cases (IVA tasas, IEPS, descuentos, redondeos) — this is the highest-risk port; credits ledger concurrency tests (parallel reserve/commit/refund); idempotency matrix.
- **Contract**: `PacProvider` tests against recorded SW fixtures (nock) + nightly live smoke against the SW sandbox (`services.test.sw.com.mx`, team test token).
- **E2E**: full flows on staging with SW sandbox (test token) + Stripe test: PUE nominal, global with `informacionGlobal`, PPD + N REPs, cancel 02, cancel 01 + sustitución, credit purchase, auto-recharge with SCA test card, webhook delivery to a receiver bin.
- **Load (k6, the acceptance gate for the 100-concurrent requirement)**: scenario ramping to **100 VUs all POSTing `/v1/invoices`** (test mode) against staging. Pass criteria:
  - 100% of stamps eventually succeed; **zero** double stamps; **zero** negative balances.
  - p95 synchronous response < 10 s; `202` fallbacks < 5% of requests.
  - Queue drains < 60 s after the burst; webhooks delivered < 2 min; PDFs ready < 3 min.
  - No 5xx; memory stable on api/worker pods.
- OpenAPI spec diff-checked in CI (no accidental breaking changes); Postman collection generated from the spec.

---

## 17. Phased build plan

Phases mirror RFP §7 (estimates unchanged); each adds technical tasks and a definition of done. **Phase -1 is new** and gates everything.

| Phase | Scope (technical) | DoD | Est. |
|---|---|---|---|
| **-1. Spike** | RFP §10 validations against the SW sandbox (**test token + test endpoint already available** — no account provisioning needed) **plus**: SW sandbox latency under 100-parallel stamps (their real limit), Gotenberg render of the SAT template with QR, BullMQ `waitUntilFinished` PoC, CSD-vs-FIEL detection heuristic | Written go/no-go note: SW limits, PDF fidelity, Stripe off-session incl. SCA card | 2–3 days |
| **0. Foundations** | Monorepo, CI/CD, Prisma + migrations, tenant extension, API-key auth, rate limiting, problem+json, pino/Sentry/metrics, docker-compose (dev), Coolify + Traefik deployment pipeline (staging) | `GET /health` live on staging via Traefik; auth+rate-limit tests green | 1.5–2 wk |
| **1. Merchants + CSD** | Merchants CRUD, CSD validation + envelope encryption, SW `certificates/save` integration, vigencia job + `csd.expiring` | CSD upload → SW-registered; tampered/wrong certs rejected with clear errors | 1.5–2 wk |
| **2. Emisión** | Tax engine port (+golden tests), sale schema, SW issue/json, idempotency, **stamp queue + QueueEvents path**, SAT templates → Gotenberg, zip, R2, signed URLs | PUE nominal invoice E2E: 201 with uuid; files ready; `invoice.stamped` delivered; k6 burst smoke | 3.5–4.5 wk |
| **3. Global + cancelación** | `informacionGlobal` validation, cancel endpoints, SW cancel + acuse, `cancel-poll` worker, cancellation webhooks | Global invoice stamped; cancel 01/02 lifecycle to `cancelled` with acuse URL | 1.5–2 wk |
| **4. PPD + REP** | Payments endpoint, REP 2.0 port, saldo validations, multi-doc REP, REP template, REP cancel orchestration, `rep_due` scheduler | Full credit-sale lifecycle: PPD → N REPs → liquidated; overpay rejected; `payment.stamped`/`rep_due` delivered | 1.5–2 wk |
| **5. Stripe** | Products/prices, Checkout, ledger reserve/commit/refund, monthly cap, auto-recharge off-session, Stripe webhooks, credit alerts | Buy → stamp → ledger consistent under parallel k6; cap blocks at 402; SCA recharge failure path verified | 2–3 wk |
| **6. Portal + docs + hardening** | Dashboard (client + admin), OpenAPI/Scalar publish, Postman, RLS, runbooks, **k6 100-VU acceptance**, security review vs OWASP API Top 10 | All §16 pass criteria green; portal covers §13 scope | 2–3 wk |
| **Total** | | | **~14–18.5 wk** |

---

## 18. Risks (technical additions to RFP §8)

| Risk | Mitigation |
|---|---|
| SW rate limits lower than assumed | Measured in spike; Bottleneck ceiling configurable; burst absorbed by queue with 202 degradation |
| Ambiguous stamp timeout → double timbre | §6.4 reconcile-by-folio before any retry; DLQ + manual tooling |
| Gotenberg instance becomes SPOF/bottleneck | Existing instance scales independently (stateless — add instances behind its endpoint); render is off critical path (files catch up) |
| Redis outage halts stamping (queue fabric) | Persistence/backups managed where Redis is independently deployed (AOF recommended); documented fail-closed runbook; queue state rebuilds from Postgres |
| Single VPS = single point of failure | Nightly `pg_dump` → R2 + provider snapshots; infra defined as compose files in-repo → redeploy to a fresh VPS < 1 h; acceptable RTO/RPO for v1; split services to a second VPS when volume justifies |
| Off-session recharge SCA failures strand clients at 0 credits | Threshold with margin, `credits.low` early, `recharge_failed` retries + emails, orderly block at 0 (RFP §8) |
| Test/live data mixing | `livemode` discriminator enforced by the same tenant extension; keys can only see their mode |
| Tax-engine port regressions | Golden-master suite from production cases before any refactor |

---

## 19. Open questions (need decisions)

1. **Hosting**: ✅ **decided** — VPS + Docker + Coolify + Traefik (§14). Remaining: provider + initial size in Phase 0 (start ~4 vCPU / 8 GB).
2. **SW sandbox**: ✅ test token + test endpoint (`services.test.sw.com.mx`) already available — the spike runs against it directly. Still open: real rate limits, folio-query API for §6.4 reconciliation, production token procurement timeline.
3. **`payment.rep_due` semantics**: which registered-but-unstamped payment states trigger it (finalize in Phase 4 design).
4. **Domains**: ✅ **decided** — API at `api.cfdi.express`, dashboard at `api.dash.cfdi.express`; API docs (Scalar) served from the API domain at `/docs` (§12).
5. **Operational email**: sender domain + templates for account notifications only — credit alerts, CSD expiry, webhook health, dashboard auth (reuse the cfdi.express Mailgun?). No buyer-facing email exists in scope.
6. **Secrets manager**: confirm current team standard (Doppler vs 1Password Connect).
7. **Webhook endpoints via public API** (§12) or dashboard-only for v1 — leaning dashboard-only, API in v1.1.
