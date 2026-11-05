# Secrets & bundling — review notes

## The one rule

In Next.js, anything prefixed `NEXT_PUBLIC_` is inlined into the client bundle. **It is public. Forever.**

## Checks

### 1. `service_role` anywhere near the client

```bash
grep -rn "service_role" --include="*.ts" --include="*.tsx" --include="*.js" .
```

Then **trace the import graph**. A helper that reads it can be imported into a client component three files away. `service_role` bypasses RLS — if it ships, the database is open.

### 2. Secrets in `NEXT_PUBLIC_` variables
Any key, token or credential with that prefix is public. Common offenders: OpenAI keys, Stripe secret keys, database URLs.

### 3. `.env*` in git history

```bash
git log --all --full-history -- .env .env.local .env.production
```

A file deleted in a later commit is still in history.

### 4. Server-only code reachable from the client
Check for `server-only` imports, and whether modules touching secrets are imported from `"use client"` components.

### 5. Build output and source maps
Production source maps can expose server code paths.

### 6. Logs and error surfaces
Secrets echoed into logs, error responses, or third-party error trackers.

## What to record

Per secret: where is it read, is it reachable from the client bundle, is it in history?
**The import-graph trace is the evidence** — "it lives in a server file" is not sufficient.
