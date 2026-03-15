---
name: deploy
description: Deploy the Hew worker to Cloudflare. Use when the user says "/deploy", "deploy the worker", "push to production", or wants to ship worker changes.
user_invocable: true
---

# Deploy

Deploys the Hew Cloudflare Worker to production.

## Steps

1. Run the tests first to make sure nothing is broken:
   ```bash
   cd worker && pnpm test
   ```

2. If tests pass, deploy:
   ```bash
   cd worker && pnpm exec wrangler deploy
   ```

3. If tests fail, stop and show the failures. Do not deploy broken code.
