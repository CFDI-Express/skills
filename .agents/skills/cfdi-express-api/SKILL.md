---
name: cfdi-express-api
description: >-
  Implement, deploy, and operate the CFDI Express API — a multi-tenant REST API
  for Mexican CFDI 4.0 invoicing (PUE/PPD income invoices, REP payment
  complements, cancellations) stamped through SW Sapien. Use when the user
  needs to: (1) set up or deploy the API monorepo, (2) add or modify invoice
  stamping, PDF rendering, or webhook flows, (3) work with the SW Sapien PAC
  integration, (4) manage credits/ledger, idempotency, or multi-tenant
  scoping, (5) troubleshoot stamp failures, cancellations, or queue workers,
  (6) extend the API surface (new routes, catalogs, billing features).
metadata:
  short-description: Implement and operate the CFDI Express invoicing API
---

# CFDI Express API

Multi-tenant REST API for CFDI 4.0 invoicing stamped through SW Sapien.

## Monorepo layout (pnpm workspaces)

| Path | Purpose |
|---|---|
| `apps/api` | Hono HTTP API (`@hono/zod-openapi`, problem+json, API-key auth) |
| `apps/worker` | BullMQ workers (stamp, postprocess, webhooks, cancel-poll, scheduler) |
| `apps/web` | React Router v7 dashboard + internal admin |
| `packages/db` | Prisma 7 schema, migrations, tenant-scoping extension, SAT catalog seeds |
| `packages/core` | Domain services: tax engine, credits/ledger, idempotency, webhook outbox |
| `packages/pac` | `PacProvider` interface + SW Sapien implementation + error normalization |
| `packages/pdf` | SAT HTML templates (invoice, global, REP) + Gotenberg client |
| `packages/shared` | Types, problem+json errors, env parsing (zod), constants |

## Core concepts

### Stamping pipeline (queue-mediated synchronous)

`POST /v1/invoices` → auth + validation → idempotency gate → credit admission (atomic conditional decrement) → enqueue stamp job → `await job.waitUntilFinished(20s)` → 201 with `uuid` or 202 `status:"stamping"` on timeout.

Worker chain: `stamp` (SW Sapien) → on success: persist XML + uuid, commit credit, chain `postprocess` → PDF via Gotenberg → zip → upload to R2 → webhook outbox.

Key rule: auto-retry ≤3 only on **pre-send** transport failures. Ambiguous post-send failures → `stamp_failed` + `needs_reconciliation` flag; never retry after send without reconciliation.

### Tenant scoping

Every tenant table carries `accountId` + `livemode`. A Prisma client extension enforces scope on every query. `sk_test_*` → SW sandbox + `livemode:false`; `sk_live_*` → production SW + `livemode:true`.

### Credits (reserve → commit/refund)

Admission: `UPDATE "Account" SET "creditsBalance" = "creditsBalance" - 1 WHERE id = $1 AND "creditsBalance" >= 1`. Stamp success → commit (`CreditLedger` `consume`); failure → refund (`+1`, ledger `refund`).

### Idempotency

`Idempotency-Key` header on all POSTs. `(accountId, key)` claimed atomically before side effects:
- completed + same requestHash → replay stored response (`Idempotency-Replayed: true`)
- different requestHash → `422 idempotency_key_reuse`
- in_flight → `409 request_in_flight`

Rows expire after 72 h. Short Redis lock around stamp enqueue collapses simultaneous duplicates.

## Error taxonomy (`application/problem+json`)

Stable codes: `validation_error` (with SAT field pointers), `sat_rejected`, `insufficient_credits`, `monthly_limit_reached`, `pac_unavailable`, `idempotency_key_reuse`, `request_in_flight`, `rate_limited`. SW/SAT errors normalized once in `packages/pac`; routes never parse vendor payloads.

## API surface (v1)

| Resource | Endpoints |
|---|---|
| Invoices | `POST /v1/invoices` · `GET /v1/invoices` · `GET /v1/invoices/{id}` · `POST /v1/invoices/{id}/cancel` |
| Payments (REP) | `POST /v1/invoices/{id}/payments` · `GET /v1/payments/{id}` · `POST /v1/payments/{id}/cancel` |
| Merchants | `GET/POST /v1/merchants` · `GET/PATCH/DELETE /v1/merchants/{id}` · `POST/PUT /v1/merchants/{id}/csd` |
| Catalogs | `GET /v1/catalogs/{products,units,usos-cfdi,regimenes,formas-pago}` |
| Webhooks | `GET/POST /v1/webhook_endpoints` · `GET/PATCH/DELETE /v1/webhook_endpoints/{id}` |
| Billing | `GET /v1/balance` · `GET /v1/usage?month=` |
| Meta | `GET /health` · `GET /ready` · `GET /docs` · `GET /openapi.json` |

## Common tasks

### Add a new route

1. Define zod schema in `apps/api/src/routes/` (use `@hono/zod-openapi`)
2. Add Prisma queries through `packages/core` service, never raw from routes
3. Add to OpenAPI tags; docs regenerate automatically at `/docs`
4. Write unit tests (Vitest) and E2E if it touches stamping/credits

### Work with the tax engine

The tax engine lives in `packages/core/tax`, ported from the Shopify app's `InvoiceData.server.ts`. Input = generic line items + receiver + flags; output = CFDI 4.0 JSON for SW `issue/json`.

Before modifying: run golden-master tests from production payloads. The calculation core is high-risk; no calculation changes without green golden tests.

### Debug a stamp failure

1. Check `Invoice.status` and `stampFailedReason` in DB
2. Check `packages/pac` logs for SW response shapes and normalized error codes
3. If `needs_reconciliation` is set: query SW by folio (serie/folio per merchant) before any retry
4. Check `CreditLedger` for refund/commit state
5. Check BullMQ DLQ for the stamp queue

### Add a SAT catalog

Import from SAT-published CSVs into `SatProduct`/`SatUnit` tables. Use trigram `ILIKE` search at v0 scale. Seeds live in `packages/db/prisma/seed/`.

### Extend the PAC interface

All SW contact is behind `PacProvider` in `packages/pac`. To add a new PAC: implement the interface, normalize errors into the taxonomy, and swap via env config. Routes must never see vendor payloads.

## Security notes

- CSD at rest: envelope encryption (per-merchant AES-256-GCM DEK + KEK)
- API keys: SHA-256 at rest, prefix lookup + constant-time compare
- Rate limiting: per-key sliding window in Redis (default 300 rpm)
- Webhooks: HMAC-SHA256 signing, timestamp tolerance 5 min, auto-disable after 100 consecutive failures

## Deployment

Self-hosted VPS + Docker + Coolify + Traefik. App containers only in prod compose; backing services (Postgres, Redis, Gotenberg, R2) are external endpoints via env vars.

Dev: `docker compose -f infra/docker-compose.yml up -d postgres redis gotenberg`

CI: typecheck → lint → test → build images → push GHCR → Coolify deploy.

## Reference docs

- **API v0 1-week plan**: See [references/api-v0-1-week-plan.md](references/api-v0-1-week-plan.md) for the sprint scope, 15-capability checklist, and day-by-day build plan.
- **Technical architecture**: See [references/technical-architecture.md](references/technical-architecture.md) for full architecture: phased build plan, data model, queue topology, webhook design, security, and deployment.
