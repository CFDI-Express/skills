---
name: cfdi-express-api
description: >-
  Integrate the CFDI Express REST API to issue Mexican CFDI 4.0 documents from
  any system: PUE/PPD/global invoices, credit notes, payment complements (REP /
  Pagos 2.0), payroll receipts (nómina / Nómina 1.2), CFDI de Retenciones 2.0
  for digital platforms (plataformas tecnológicas), cancellations with SAT
  acknowledgment, issuer (RFC/CSD) management, SAT catalogs, prepaid balance
  and signed webhooks. Stripe-style conventions. Use when the user needs to
  stamp, list, fetch or cancel CFDIs (including recibos de nómina and
  constancias de retenciones), register issuers or upload CSD certificates,
  search SAT product/unit/usos/regimen/nómina/retenciones codes, check balance
  and usage, or receive webhooks about stamped documents against
  api.cfdi.express.
metadata:
  short-description: Integrate the CFDI Express invoicing REST API
---

# CFDI Express API

REST API that stamps CFDI 4.0 documents against the SAT from any system that
speaks HTTPS: PUE, PPD and global invoices, credit notes, payment complements
(REP), payroll receipts (nómina), retenciones for digital platforms and
cancellations with acknowledgment. Stripe-style conventions: secret keys,
idempotency keys, cursor pagination, RFC 7807 errors, signed webhooks.

$1 MXN per stamp, prepaid balance, free unlimited sandbox.

## Authoritative sources — always fetch exact schemas, never guess

- OpenAPI spec (generate clients from it): https://api.cfdi.express/openapi.json
- Interactive docs (try endpoints live): https://api.cfdi.express/docs
- Agent-friendly reference index: https://cfdi.express/docs/api/llms.txt —
  includes a section per feature release (serie, fecha, pedimentos, foreign
  receptor, retenciones, lineDecimals…)
- Per-endpoint markdown: append `/llms.txt` to any guide URL,
  e.g. https://cfdi.express/docs/api/timbrar-factura/llms.txt

Read the relevant `llms.txt` page before writing code against an endpoint.
This skill is the map; those pages hold every field and SAT rule.

## Base URL and auth

- Base: `https://api.cfdi.express`, all v1 endpoints under `/v1`. HTTPS only.
- Every request: `Authorization: Bearer sk_test_...` or `sk_live_...`
- `sk_test_` stamps in the PAC sandbox for free; `sk_live_` issues real
  CFDIs. Modes never share balance, issuers or documents (`livemode` field).
- Keys are secret: server-side only, never browsers or mobile apps.
- Each account is a workspace that can have several members; an API key
  belongs to one workspace, so integrations need no extra header.
- Get keys at https://dash.cfdi.express

```bash
curl https://api.cfdi.express/v1/balance \
  -H "Authorization: Bearer sk_test_..."
```

## Core rules

- **Idempotency**: stamping/cancel endpoints require an `Idempotency-Key`
  header with a stable value per logical operation (your order id, etc.).
  Replays return the original response with `Idempotency-Replayed: true`.
  Same key + different body → `422`; concurrent same key → `409`.
- **Sync with fallback**: stamping waits for the SAT (typically a few
  seconds) and returns `201`. Under load it returns `202` with
  `status: "stamping"` — poll `GET` on the resource or use webhooks.
- **Errors**: `application/problem+json` with stable `code`:
  `validation_error` (400, includes `errors[]` with field paths and, for some
  rules, a `subcode` such as `overpay`, `fecha_out_of_range`,
  `merchant_archived`, `csd_missing`, `cancel_reps_first`),
  `unauthorized` (401), `forbidden` (403), `not_found` (404),
  `insufficient_credits` / `monthly_limit_reached` (402),
  `request_in_flight` (409), `idempotency_key_reuse` / `sat_rejected` (422),
  `rate_limited` (429, honor `Retry-After`),
  `pac_unavailable` / `billing_unavailable` (503, safe to retry with the same
  key), `internal_error` (500).
- **Pagination**: cursor-based. `limit` (max 100) + `starting_after=<last id>`
  while `has_more` is `true`.
- **Limits**: 300 req/min per account; 1 MB body; 1,000 concepts per invoice.
- **Signed URLs**: XML/PDF/ZIP/acknowledgment URLs expire after 15 minutes —
  refetch the resource when you need fresh ones.
- **Balance**: prepaid, in MXN centavos (`GET /v1/balance` returns
  `balanceCentavos` and `timbresRemaining`). Each stamp reserves its price and
  commits it on UUID issuance; SAT rejection or PAC failure auto-refunds.
  Cancellations also consume one stamp.

## Integration flow

1. Register an issuer: `POST /v1/merchants` (RFC, régimen fiscal, ZIP, serie)
2. Upload its CSD: `POST /v1/merchants/{id}/csd` (.cer/.key base64 + password)
3. Stamp: `POST /v1/invoices` (the tax engine computes IVA/IEPS/retentions)
4. Deliver XML/PDF via signed URLs from `GET /v1/invoices/{id}`, or receive
   them in an `invoice.stamped` webhook

## Invoices — options worth knowing

All on `POST /v1/invoices`; most also on `POST /v1/credit_notes`:

- **PUE vs PPD**: PUE needs the real `formaPago`; PPD requires `formaPago`
  `99` and leaves a `saldoPendiente` for payment complements.
- **Público en general (global)**: receiver `XAXX010101000`, name
  `PUBLICO EN GENERAL`, `usoCfdi` `S01`, `regimenFiscal` `616`, zip = the
  issuer's, plus `informacionGlobal`. `XAXX010101000` with the customer's own
  name is a nominativa invoice and takes no `informacionGlobal`.
- **Foreign receptor**: `XEXX010101000` with `receiver.residenciaFiscal`
  (c_Pais, never `MEX`) and `receiver.numRegIdTrib`; `S01`, `616`, issuer zip.
- `serie` / `folio`: per-document serie (1–25 chars, no `|`) and your own
  folio; the internal folio sequence is shared across series.
- `fecha`: back-date up to 70 hours, Mexico City local time
  `YYYY-MM-DDTHH:mm:ss`.
- `items[].pedimentos` (InformacionAduanera) and `items[].cuentasPrediales`
  (CuentaPredial, for leases).
- `lineDecimals: 6`: six-decimal concepto amounts so tax-inclusive prices
  close exactly (document totals still round to 2).
- `logoUrl`: per-document PDF logo overriding the issuer's.

Credit notes (`POST /v1/credit_notes`) relate to the original via
`relatedInvoiceId` (stamped here; receiver copied) or `relatedUuid` +
`receiver` (stamped elsewhere). They are invoice resources with
`kind: "credit_note"`.

## Payment complements (REP)

For PPD invoices, stamp a REP per payment received:
`POST /v1/invoices/{id}/payments` with `monto`, the real `formaPago` (not
`99`) and optional `fechaPago`. `numParcialidad`, `saldoAnterior` and
`saldoInsoluto` are computed server-side; overpaying returns `400` with
`subcode: "overpay"`. A PPD invoice with live REPs can't be cancelled until
its REPs are (`cancel_reps_first`); cancelling a REP restores the balance.

## Payroll (nómina)

`POST /v1/nominas` stamps one recibo de nómina (CFDI N + complemento Nómina
1.2) per employee and pay period — send `employee`, `percepciones`,
`deducciones` (ISR = tipo 002), `otrosPagos` (subsidio al empleo = tipo 002)
and the API derives every SAT total; claves come from
`GET /v1/catalogs/nomina/{catalog}`. Never invent registro patronal, NSS or
ISR amounts.

## Retenciones (digital platforms)

`POST /v1/retenciones` stamps a CFDI de Retenciones 2.0 with the complemento
Servicios de Plataformas Tecnológicas: one constancia per provider per month
(`merchantId`, `ejercicio`, `mes`, `tipoServicio`, `receiver`, `servicios[]`
with `fecha`, `formaPago`, `precioSinIva`, `comision`). The API derives IVA,
ISR and IVA retenido from the service type, year and provider; look up
`tipoServicio` in `GET /v1/catalogs/retenciones/tipo-servicio` — never guess
it. Guide: https://cfdi.express/docs/api/timbrar-retencion/llms.txt

## Webhooks

Register an endpoint with `POST /v1/webhook_endpoints` (`url` + event types;
catalog at `GET /v1/webhook_event_types`). Events:
`{invoice,payment,nomina,retencion}.{stamped,stamp_failed,cancelled}` — credit
notes arrive as `invoice.*`. `data.object` is the same resource the `GET`
endpoint returns, with file URLs presigned per delivery attempt.

- Verify `CFDI-Signature: t=<unix>,v1=<hex>[,v1=<hex>]`:
  `hex(HMAC_SHA256(secret, "{t}.{raw body}"))` over the raw request bytes
  (re-serialized JSON won't match). Accept if any `v1` matches (two during a
  secret rotation) and reject `t` older than 5 minutes.
- Answer `2xx` within 10 seconds (redirects count as failures); do the work
  after responding.
- Dedupe on the `CFDI-Delivery` header or the event `id`; events can arrive
  out of order (compare `created`). Failed deliveries retry for ~46 hours.
- Delivery log and replays: `GET /v1/webhook_endpoints/{id}/deliveries`,
  `POST /v1/webhook_deliveries/{id}/retry`, `POST /v1/webhook_endpoints/{id}/test`.

## Endpoints

| Resource | Endpoints |
|---|---|
| Invoices | `POST /v1/invoices` · `GET /v1/invoices` · `GET /v1/invoices/{id}` · `POST /v1/invoices/{id}/cancel` |
| Credit notes | `POST /v1/credit_notes` (read, list and cancel through the invoice endpoints) |
| Payments (REP) | `POST /v1/invoices/{id}/payments` · `GET /v1/payments` · `GET /v1/payments/{id}` · `POST /v1/payments/{id}/cancel` |
| Nómina (payroll) | `POST /v1/nominas` · `GET /v1/nominas` (filters `employeeRfc`, `fechaPagoFrom/To`) · `GET /v1/nominas/{id}` · `POST /v1/nominas/{id}/cancel` |
| Retenciones | `POST /v1/retenciones` · `GET /v1/retenciones` (filters `receiverRfc`, `ejercicio`, `mes`) · `GET /v1/retenciones/{id}` · `POST /v1/retenciones/{id}/cancel` |
| Issuers | `GET/POST /v1/merchants` · `GET/PATCH/DELETE /v1/merchants/{id}` · `POST /v1/merchants/{id}/archive` · `POST /v1/merchants/{id}/unarchive` · `POST/PUT/GET/DELETE /v1/merchants/{id}/csd` · `PUT/DELETE /v1/merchants/{id}/logo` |
| SAT catalogs | `GET /v1/catalogs/{products,units,usos-cfdi,regimenes,formas-pago}` · `GET /v1/catalogs/nomina/{catalog}` · `GET /v1/catalogs/retenciones/{tipo-servicio,subtipo-servicio,forma-pago-servicio,periodicidad,tipo-pago-retencion,entidad-federativa}` |
| Webhooks | `GET /v1/webhook_event_types` · `GET/POST /v1/webhook_endpoints` · `GET/PATCH/DELETE /v1/webhook_endpoints/{id}` · `POST /v1/webhook_endpoints/{id}/rotate_secret` · `POST /v1/webhook_endpoints/{id}/test` · `GET /v1/webhook_endpoints/{id}/deliveries` · `GET /v1/webhook_deliveries/{id}` · `POST /v1/webhook_deliveries/{id}/retry` |
| Balance & billing | `GET /v1/balance` · `POST /v1/billing/checkout_sessions` · `POST /v1/billing/portal_sessions` · `GET /v1/billing/usage` · `GET /v1/billing/transactions` |
| Meta | `GET /health` · `GET /ready` |

Archived issuers are hidden from `GET /v1/merchants` by default and can't
stamp (`merchant_archived`) until unarchived. Cancellation motivo `01`
requires `folioSustitucion` (the replacing UUID).

Guide for each endpoint: https://cfdi.express/docs/api (append `/llms.txt` for markdown).

## Alternative for AI agents: MCP

Instead of raw REST, agents can connect to the MCP server at
`https://api.cfdi.express/mcp` (sandbox: `/mcp/test`): 22 tools covering
invoices, credit notes, REPs, nómina, retenciones, cancellations, queries,
catalogs and balance, plus report prompts (`cierre_de_mes`,
`cuentas_por_cobrar`). Auth via API key or OAuth 2.1; each OAuth connection
is bound to one workspace. Guide: https://cfdi.express/docs/api/mcp
