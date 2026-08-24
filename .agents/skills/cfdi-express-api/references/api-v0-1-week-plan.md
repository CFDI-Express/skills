# API v0 — 1-Week Build & Deploy Plan (migration-driven scope)

**Version:** 1.0 · **Date:** 2026-07-28 · **Author:** Acromático Development
**Companion docs:** [`technical-architecture.md`](./technical-architecture.md) (target architecture, 14–18.5 wk full build) · [`rfp-cfdi-api.md` v1.5](../../acromatico-cfdi-express/docs/rfp-cfdi-api.md) (product spec + Facturama substitution inventory, Anexo A).

---

## 1. What this plan is

A **7-day sprint** to design, build and deploy **CFDI Express API v0** — the minimal real API whose surface covers **100% of the capabilities the Shopify app consumes from Facturama today** (RFP Anexo A.7, 15/15 capabilities). The Shopify app migration itself is **out of this sprint** and happens afterwards; v0's definition of done is that nothing in the migration is blocked by the API.

This is only possible by cutting everything the migration doesn't strictly need and by lifting proven code from the Shopify app. The full architecture (Stripe billing, webhooks, dashboard, cancel-polling, hardening) is **deferred, not redesigned** — v0 keeps the target architecture's boundaries so each deferred piece drops in later without rework (§6).

### v0 in one paragraph

Multi-tenant-schema Hono API + BullMQ worker + Postgres/Prisma + Redis + Gotenberg + R2, deployed via Docker/Coolify/Traefik, that: authenticates with API keys, manages merchants with validated+encrypted CSDs registered in SW Sapien, stamps PUE (nominativa y global) and PPD invoices through a queue-mediated synchronous path with persistent idempotency, stamps REP payment complements, cancels invoices/REPs and stores the acuse, renders its own SAT PDFs, serves files as signed URLs, and answers SAT catalog queries from local data. Credits exist (ledger + reserve/commit/refund) but are **seeded, not sold** — no Stripe in v0.

### Assumptions (if any fails, the week slips)

1. **1 senior dev, full-time, 7 calendar days**, AI-assisted, no other commitments.
2. SW Sapien **sandbox token + test endpoint** (`services.test.sw.com.mx`) available on day 1 (already true per architecture §19).
3. Existing Gotenberg instance reachable; R2 buckets + VPS/Coolify/Traefik + DNS for `api.cfdi.express` ready on day 1.
4. SW **production** token is *not* required for the v0 deploy — v0 ships fully functional in `test` mode; `live` mode is gated on token procurement (§8 risk R5).
5. SAT product/unit catalogs imported from the SAT-published CSVs (public data).

---

## 2. Scope: the contract

### 2.1 IN — the 15 capabilities of RFP Anexo A.7 (acceptance checklist)

Every row must be callable against the deployed v0 (test mode, SW sandbox) at the end of day 7:

| # | Capability (RFP A.7) | v0 endpoint | Day |
|---|---|---|---|
| 1 | Stamp income invoice (PUE/PPD) | `POST /v1/invoices` | 3, 5 |
| 2 | Stamp REP | `POST /v1/invoices/{id}/payments` | 5 |
| 3 | PDF of invoice/REP | signed URLs on `GET /v1/invoices/{id}` / `GET /v1/payments/{id}` | 4, 5 |
| 4 | Stamped XML | same (from SW stamp response, no extra fetch) | 3 |
| 5 | Zip PDF+XML | `zipUrl` on the resource | 4 |
| 6 | Cancel invoice (motivos 01–04 + `folioSustitucion`) | `POST /v1/invoices/{id}/cancel` | 5 |
| 7 | Cancel REP | `POST /v1/payments/{id}/cancel` | 5 |
| 8 | Cancellation acuse | `cancellation.acuseUrl` on the resource | 5 |
| 9 | CSD upload | `POST /v1/merchants/{id}/csd` | 2 |
| 10 | CSD update | `PUT /v1/merchants/{id}/csd` | 2 |
| 11 | CSD vigencia | `GET /v1/merchants/{id}` (metadata from SW) | 2 |
| 12 | Units catalog | `GET /v1/catalogs/units?keyword=` | 6 |
| 13 | Products/services catalog | `GET /v1/catalogs/products?keyword=` | 6 |
| 14 | Opaque durable id | `id` (cuid2) on every resource; `uuid` as data; `pacDocumentId` internal | 1 |
| 15 | Cancellation status | `cancellation.status` (as returned by SW at cancel time; polling deferred) | 5 |

Plus the cross-cutting mechanics the migration depends on: API-key auth (`sk_test_`/`sk_live_`, SHA-256 at rest), `livemode` discriminator everywhere, persistent `Idempotency-Key` (replay / 409 / 422-reuse), `application/problem+json` errors with stable codes, cursor pagination, `GET /health` + `/ready` + `/docs` (Scalar) + `/openapi.json`.

### 2.2 OUT — deferred to fast-follow (each maps to an architecture phase)

| Deferred | Why it's safe to cut for v0 | Fast-follow |
|---|---|---|
| **Stripe billing** (Checkout, auto-recharge, monthly cap, Stripe webhooks) | v0's only consumer is our own Shopify app; credits are seeded. `CreditLedger`, `creditsBalance`, reserve/commit/refund and the `402` admission check ship in v0 so the semantics don't change later — only the funding source is missing. | Arch. phase 5 |
| **Webhook subsystem** (outbox, signing, retries, endpoints CRUD) | The app today works synchronously; it will poll `GET /v1/invoices/{id}` until `filesStatus:"ready"` during migration. The stamp→postprocess job chain keeps an explicit seam where outbox rows get written later. | Arch. §8 (pre-migration or during) |
| **Dashboard / portal** | One tenant (us); merchants/keys/CSDs seeded and managed via API/Prisma Studio. | Arch. phase 6 |
| **cancel-poll worker** (SAT receptor-acceptance tracking ≤72 h) | The app doesn't query cancellation status today (A.7 row 15 is a net improvement we partially ship: status at cancel time). | Arch. phase 3 |
| **Schedulers** (`payment.rep_due`, `csd.expiring`, SW balance monitor) | Alerting niceties, not migration blockers. | Arch. phases 3–4 |
| **k6 100-VU acceptance, RLS, runbooks, OWASP review** | v0 load = one tenant's production traffic; burst design ships (queue-mediated stamping) but formal acceptance is deferred. | Arch. phase 6 |
| Operational email (credit/CSD/webhook alerts) | Nothing to alert yet — no credits sold, no webhooks. | With their subsystems |

**This list is the anti-scope-creep contract for the week.** Anything not in §2.1 waits, no matter how small it looks.

### 2.3 What v0 is NOT (be honest with ourselves)

- Not production-hardened for unknown third-party tenants (no RLS, basic rate limiting, no security review).
- Not load-certified (the 100-concurrent *design* ships; the k6 *acceptance* does not).
- PDF fidelity is "SAT-complete and reviewed against Anexo 20 samples", not pixel-perfect; expect template iteration in week 2.
- Multi-document REP (one payment settling several invoices) ships if day 5 holds; otherwise single-document REP + explicit `validation_error` for multi-doc (§7 cut list).
- SAT edge validations beyond the migration's needs are documented-not-handled (e.g., exotic régimen↔uso combinations fail at SW and surface as `sat_rejected`).

---

## 3. Reuse inventory (why 1 week is feasible)

Lift-and-shift sources in the Shopify app (`acromatico-cfdi-express`), audited 2026-07-28:

| v0 component | Source | Reuse |
|---|---|---|
| Tax engine (IVA multi-tasa, IEPS, descuentos, redondeos, item building) | `app/models/InvoiceData.server.ts` (858 ln) | Port: strip Shopify order lookups → generic line items input; keep calculation core untouched. **Golden tests from production payloads before any refactor.** |
| REP 2.0 builder (`Complemento.Payments`, related docs, tax proration) | `app/routes/api.complementos.ts` (702 ln) | Port the payload-construction core; replace Facturama envelope with SW `issue/json`. |
| Zip PDF+XML packaging | `app/models/Cfdi.server.ts` (JSZip block) | Lift nearly verbatim. |
| Idempotency semantics (claim/complete/release, duplicate replay) | `app/models/CfdiIdempotency.server.ts` (160 ln) | Port Redis-claim → Postgres persistent key + short Redis lock (arch. §10). |
| CSD upload/validation flows | `app/routes/api.csd.ts` (395 ln) | Port validation logic; add envelope encryption + SW `certificates/save`. |
| SAT catalog seeds (régimenes, usos CFDI, formas de pago) | `prisma/seed/*.json`, `prisma/schema.prisma` (`RegimenFiscal`, `SatUsoCfdi`, `SatPaymentMethod`) | Migrate seeds as-is; add `SatProduct`/`SatUnit` from SAT CSVs. |
| BullMQ + ioredis wiring patterns | `app/lib/queue.server.ts`, `app/lib/services/report-worker.server.ts` | Copy the connection/queue setup idioms (bullmq@5). |
| Operational know-how | production Facturama error catalogue | Feeds `packages/pac` error normalization (arch. §11.4). |

**Net-new in v0** (no reuse — where the week's risk lives): SW Sapien integration (never called before), own SAT PDF template (Facturama rendered PDFs until now), Hono/zod-openapi skeleton, deployment pipeline, SAT product/unit catalog import.

---

## 4. The 7-day plan

Monorepo per architecture §4 but trimmed: `apps/api`, `apps/worker`, `packages/{db,core,pac,pdf,shared}`. No `apps/web`. pnpm workspaces, TypeScript strict.

### Day 1 — Foundations + SW sandbox smoke (the compressed spike)

**Build:**
- Monorepo scaffold, base tsconfig, eslint/prettier, pnpm scripts (`dev`, `check`, `test`).
- Prisma schema (v0-trimmed from arch. §5): `Account`, `ApiKey`, `Merchant`, `Csd`, `Invoice`, `Payment` + `PaymentDocument`, `Cancellation`, `CreditLedger`, `UsageMonthly`, `IdempotencyKey`, `AuditLog`, SAT catalogs. `accountId` + `livemode` on every tenant table from line one. Tenant-scoping Prisma extension.
- `infra/docker-compose.yml` (dev): Postgres 16, Redis 7, Gotenberg, Mailpit(optional), api, worker.
- Hono + `@hono/zod-openapi` skeleton: auth middleware (prefix lookup + constant-time SHA-256 compare), request-id, pino, problem+json helper with the arch. §11.4 code taxonomy, `/health`, `/ready`, `/docs`, `/openapi.json`.
- Seed script: SAT catalogs (régimenes/usos/formas-pago from the app's seed JSONs) + 1 account + `sk_test_`/`sk_live_` keys + seeded credit balance (10⁶).
- **SW smoke script** (`scripts/sw-smoke.ts`, not a route — this is RFP §10 compressed): `certificates/save` with the test CSD → `issue/json/v4` PUE nominativa → issue global with `InformacionGlobal` → cancel motivo 02 → fetch acuse → `balance`. Log raw response shapes and error shapes; measure stamp latency ×20.

**DoD:** smoke script stamps + cancels end-to-end against the SW sandbox; auth returns 401/200 correctly; `docker compose up` boots the stack; CI runs `check` + `test`.

### Day 2 — `packages/pac` + Merchants & CSD

**Build:**
- `PacProvider` interface (`uploadCsd`, `stamp`, `cancel`, `balance`) + SW implementation + error normalization into the v0 taxonomy. v0 retry rule (simplified arch. §6.4): auto-retry ≤3 **only** on pre-send transport failures; ambiguous post-send failures go to `stamp_failed` with a `needs_reconciliation` flag (manual) — no auto-retry after send in v0.
- CSD pipeline: local validation (vigencia window, cer/key correspondence, RFC extraction, CSD-not-FIEL heuristic — confirmed by day-1 smoke) → envelope encryption (per-merchant AES-256-GCM DEK + master key from env) → SW `certificates/save` → persist metadata.
- Routes: `GET/POST /v1/merchants`, `GET/PATCH/DELETE /v1/merchants/{id}`, `POST/PUT /v1/merchants/{id}/csd`, `GET /v1/merchants/{id}/csd`.

**DoD:** test CSD uploaded via API is registered in SW (verified by stamping against it); tampered/expired/mismatched certs rejected with clear `validation_error` subcodes; secrets verified encrypted at rest.

### Day 3 — Tax engine + emission core (PUE nominativa)

**Build:**
- Port `InvoiceData.server.ts` → `packages/core/tax`: input = generic line items + receiver + flags (`ivaZeroItems`, prices-include-tax, discounts); output = CFDI 4.0 JSON for SW `issue/json`. Golden-master tests from ≥6 production payloads (IVA 16/8/0, IEPS, descuento, redondeo edge).
- Sale zod schema + validations: RFC/CP formats, régimen↔uso CFDI, PUE requires real `formaPago`, global ⇒ PUE only (day 4 enforces), PPD ⇒ `formaPago 99`.
- Stamp path per arch. §6.1: idempotency gate (Postgres claim, replay/409/422) → credit admission (atomic conditional `UPDATE … WHERE balance >= 1`, reserve) → `Invoice(stamping)` + enqueue → `QueueEvents.waitUntilFinished(20s)` → success: persist XML + uuid, commit credit → **201** `{ id, status:"stamped", uuid, files:{status:"processing"} }`; SAT 4xx → refund + 422 `sat_rejected`; timeout → 202 (kept, it's nearly free).
- `stamp` worker (concurrency 25, Bottleneck ceiling from day-1 measurement) chains `postprocess` job (stub until day 4).

**DoD:** nominativa PUE stamps E2E via API on sandbox with uuid + XML persisted; duplicate `Idempotency-Key` replays the same response; concurrent duplicate → 409; SAT rejection → 422 + credit refunded; golden tax tests green.

### Day 4 — PDF, files & global invoices

**Build:**
- `packages/pdf`: SAT invoice template (HTML/CSS: emisor/receptor fiscal block, conceptos, impuestos, totales, **QR** with SAT verification URL (uuid+rfcEmisor+rfcReceptor+total+sello last-8), cadena original del complemento de certificación, sellos, fecha y no. de certificado SAT, merchant logo) + Gotenberg client (multipart `/forms/chromium/convert/html`, 30 s timeout, 3 retries).
- `postprocess` worker for real: render PDF → zip (ported JSZip block) → upload XML/PDF/zip to R2 → `filesStatus:"ready"`. Signed URLs (15 min TTL) generated on demand at read time.
- `GET /v1/invoices/{id}` (full resource: receiver, totals, `cancellation`, `saldoPendiente`, signed file URLs) + `GET /v1/invoices` with filters (merchant, dates, status, uuid, método) + cursor pagination.
- `informacionGlobal` validation (periodicidad/meses/año combinations per SAT; global ⇒ PUE, receptor `XAXX010101000`) → global invoice E2E.

**DoD:** after stamping, `GET` returns working `pdfUrl/xmlUrl/zipUrl`; PDF reviewed side-by-side against an Anexo 20 sample and a Facturama-rendered production PDF; global invoice stamps with `InformacionGlobal`.

### Day 5 — PPD + REP + cancellation

**Build:**
- PPD invoice path (método `PPD`, forma `99`, `saldoPendiente` initialized to total).
- Port REP builder from `api.complementos.ts` → `POST /v1/invoices/{id}/payments`: parcialidad sequencing, saldo anterior/insoluto coherence, overpay rejection (`validation_error`), currency check; multi-doc via `PaymentDocument` join if the day holds. Same stamp→postprocess chain; REP PDF template variant; `Payment` resource: `GET /v1/payments/{id}`.
- Cancellation: `POST /v1/invoices/{id}/cancel` + `POST /v1/payments/{id}/cancel` — validate motivo 01–04, `folioSustitucion` required iff motivo 01, PPD-with-REPs rule (must cancel REPs first); SW cancel → store acuse in R2 → `cancellation { motivo, folioSustitucion, status, acuseUrl }` on the resource; `202` semantics kept even though SW answers synchronously.

**DoD:** full credit-sale lifecycle on sandbox: PPD → REP parcialidad 1 → REP final (insoluto 0) → liquidated; overpay rejected; cancel 02 returns acuse URL; cancel 01 + folio works; cancel of PPD with live REPs rejected with the orchestration error.

### Day 6 — Catalogs, guardrails, E2E suite, staging deploy

**Build:**
- SAT product/unit catalog import (SAT CSVs → `SatProduct`/`SatUnit`, trigram-ish `ILIKE` search is fine at v0 scale) + `GET /v1/catalogs/{products,units,usos-cfdi,regimenes,formas-pago}`.
- Basic guardrails: per-key sliding-window rate limit (300 rpm default, Redis), 1 MB body limit, `AuditLog` writes on mutations.
- E2E suite (Vitest, SW sandbox): the five flows — PUE nominativa, global, PPD+REPs, cancel 02, cancel 01+sustitución — plus auth/idempotency/credits matrix.
- Dockerfiles (`api`, `worker`), CI: typecheck → lint → test → build → push GHCR; Coolify project + Traefik routing for staging.

**DoD:** staging live at `api-staging.cfdi.express` (test mode; `api.cfdi.express` is reserved for the Day-7 production deploy — architecture §20-4), `/docs` (Scalar) + `/openapi.json` served; E2E suite green in CI against sandbox.

### Day 7 — Production deploy, burst smoke, buffer

**Build:**
- Production Coolify deploy (same images; prod DB/Redis endpoints, prod R2 bucket); seed production account + keys; runbooks-lite in `docs/` (SW outage behavior, refund/reconciliation manual procedure).
- k6 **burst smoke** (not the full acceptance): 30–50 concurrent `POST /v1/invoices` on staging → verify zero double-stamps, zero negative balances, queue drains, 202 fallback behaves.
- **Buffer** (this day absorbs the week's slips — historically: PDF fidelity, SAT validation surprises, PPD edge cases).

**DoD:** v0 deployed to production (test mode; `sk_live_` present but SW-side gated on the production token, §8-R5); §2.1 checklist executed end-to-end against the deploy; migration-readiness note written.

---

## 5. After the week: migration (summary, not this sprint's work)

Executed against a stable v0, roughly in this order (detail planned when scheduled):

1. **API client** in the Shopify app (`app/lib/cfdi-api.server.ts`): Bearer key, idempotency keys per (shop, order), problem+json parsing, poll-until-`filesStatus:"ready"` helper (until webhooks land).
2. **Swap the 9 Facturama call sites in 6 files** (RFP A.1–A.5): `Cfdi.server.ts:428` (stamp), `api.complementos.ts:355` (REP), `Cfdi.server.ts:462/465` + `api.complementos.ts:396/408` (PDF/XML — now from resource URLs), `api.acuse.ts:88` (acuse), `Cfdi.server.ts:112` + `api.complementos.ts:614` (cancels), `api.csd.ts:106/:258/:290` (CSD), `api.catalogos.unidades.ts:12` / `api.catalogos.productos.ts:16` (catalogs). Delete the 5 duplicated Basic-auth/env-switch blocks (A.6.3).
3. **Identity mapping**: `facturama_id` stays for historical documents (old records keep reading/acusing from Facturama during a transition window — credentials go read-only); new documents store the API's opaque `id` + `uuid`.
4. **Behavior changes to absorb**: cancellation becomes `202`-style (status on resource), files come from URLs not base64, error shapes change (map `sat_rejected`/`validation_error` to the app's current messages).

---

## 6. Forward-compatibility rules (don't paint v0 into a corner)

1. `livemode` + `accountId` on every resource; tenant extension enforces scope on every query — even with one tenant.
2. Credit semantics exactly per arch. §10 (reserve → commit/refund, conditional decrement, ledger rows) — only the Stripe funding source is absent.
3. The postprocess job writes `filesStatus` transitions in one place; the future webhook outbox insert goes in that same transaction (arch. §8.2).
4. Opaque `id` everywhere; `pacDocumentId` never leaves the server; `uuid` is plain data.
5. All SW contact inside `packages/pac`; routes never see vendor payloads.
6. Migrations expand-only; nothing in v0's schema requires a breaking change for phases 5–6 (Stripe fields, webhook tables are additive).

---

## 7. Cut list (pre-agreed, if the week slips)

In order of sacrifice:

1. Multi-document REP → single-document only; multi-doc returns `validation_error` with a clear message.
2. REP PDF template polish → reuse the invoice template skeleton with REP data blocks.
3. Full SAT product catalog (~60k claves) → top ~5k most-used + note in docs; units catalog is small, keep full.
4. Per-account rate-limit config → single env-global limit.
5. `/v1/catalogs/*` beyond `products`/`units` → `usos-cfdi`/`regimenes`/`formas-pago` are tiny static seeds, but they're the next cut before touching 1–4.

Never cut: idempotency, credit refund-on-failure, tax-engine golden tests, SW error normalization, encryption at rest.

---

## 8. Risks specific to the 1-week compression

| # | Risk | Mitigation |
|---|---|---|
| R1 | **SW integration never tested** — payload shape, error codes, certificate flow unknowns | Day-1 smoke script fronts all of it; if `issue/json` shape differs materially from docs, days 2–3 re-plan immediately, cut list activates |
| R2 | **PDF SAT fidelity** — biggest net-new artifact; Anexo 20 details (QR contents, cadena original, sello formatting) are fiddly | Start template from a known-good open SAT layout; side-by-side review vs a Facturama production PDF on day 4; accept "complete, not pixel-perfect" for v0 |
| R3 | **Tax-engine port regressions** (Shopify-coupled inputs) | Golden-master tests from production payloads written *before* the refactor; no calculation changes during the port |
| R4 | **Ambiguous stamp failure → double timbre** | v0 rule: no auto-retry after send; `stamp_failed` + `needs_reconciliation` flag + documented manual procedure; full reconcile-by-folio in fast-follow |
| R5 | **SW production token not procured in time** | v0 ships test-mode-complete; `sk_live_` works the moment the token lands in env vars — no code change; procurement starts day 1 in parallel |
| R6 | **Single-dev single-week fragility** (illness, blocker day) | Day 7 is declared buffer; cut list pre-agreed (§7); daily DoD gates make slippage visible by day 3 |
| R7 | SAT catalog data wrangling eats day 6 | Import script is mechanical; fallback = §7.3 |

---

## 9. Definition of done (v0)

- [ ] All 15 §2.1 capabilities verified against the production deploy in test mode (SW sandbox).
- [ ] E2E suite green in CI: PUE nominativa, global, PPD+N REPs, cancel 01/02, idempotency/credits matrix.
- [ ] k6 burst smoke: no double stamps, no negative balances, sane drain.
- [ ] `/docs` + `/openapi.json` public; seed onboarding doc for the future migration.
- [ ] Deploys reproducible: images in GHCR, Coolify projects for staging + prod, `prisma migrate deploy` as pre-deploy task.
- [ ] Fast-follow backlog issues opened for every §2.2 row, mapped to architecture phases.
