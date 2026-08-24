# CFDI Express Skills

Public AI skills for implementing and operating the [CFDI Express API](https://github.com/cfdi-express/cfdi-express-api) — a multi-tenant REST API for Mexican CFDI 4.0 invoicing.

## Usage

These skills are designed for [OpenCode](https://opencode.ai/) and compatible agents. To use them, copy or symlink the skill directories into your agent's skill path:

```bash
# For OpenCode
mkdir -p ~/.config/opencode/skills
cp -r .agents/skills/cfdi-express-api ~/.config/opencode/skills/

# Or symlink for development
ln -s "$(pwd)/.agents/skills/cfdi-express-api" ~/.config/opencode/skills/cfdi-express-api
```

## Skills

### `cfdi-express-api`

Guides agents through:
- Setting up and deploying the API monorepo
- Working with the stamping pipeline (PUE/PPD invoices, REP payment complements, cancellations)
- SW Sapien PAC integration and error normalization
- Credits/ledger, idempotency, and multi-tenant scoping
- Troubleshooting stamp failures, queue workers, and webhooks
- Extending the API surface (routes, catalogs, billing)

Includes reference documentation for the full technical architecture and v0 build plan.

## License

MIT
