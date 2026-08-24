# CFDI Express Skills

AI agent skill for integrating the [CFDI Express API](https://cfdi.express/docs/api) — a REST API for Mexican CFDI 4.0 invoicing (PUE/PPD/global invoices, credit notes, REP payment complements, cancellations) with Stripe-style conventions.

Install it and your AI assistant gets the full context to implement the API in your system: auth, idempotency, error handling, the endpoint map and pointers to the live docs.

## Install

```bash
npx skills add CFDI-Express/skills
```

Or install only this skill / target a specific agent:

```bash
npx skills add CFDI-Express/skills --skill cfdi-express-api
npx skills add CFDI-Express/skills -g -a claude-code
```

Works with Claude Code, Cursor, Codex, OpenCode and any agent supported by [skills](https://github.com/vercel-labs/skills).

## Skills

### `cfdi-express-api`

Everything an integrator needs:

- Base URL, bearer-key auth and `sk_test_` vs `sk_live_` modes
- Idempotency keys, cursor pagination, RFC 7807 error codes, rate limits
- The 4-step integration flow (issuer → CSD → stamp → deliver XML/PDF)
- Full v1 endpoint map for invoices, credit notes, REPs, issuers, SAT catalogs and billing
- Links to the authoritative sources so agents fetch exact schemas instead of guessing:
  - OpenAPI spec: https://api.cfdi.express/openapi.json
  - Interactive docs: https://api.cfdi.express/docs
  - Per-endpoint markdown: https://cfdi.express/docs/api/llms.txt

## License

MIT
