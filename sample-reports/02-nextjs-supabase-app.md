# Sample report — AI-generated Next.js + Supabase app

*Anonymized. Client identifiers replaced with `tenant-a`, `example.com`, `[REDACTED]`.*

**Scope:** pre-launch review of a Next.js 15 (App Router) + Supabase application built primarily with an AI coding tool. ~4,800 lines. Read-only repository access. No production access.

## Threat model

- **Assets** — user profiles (email, name), uploaded documents, subscription state, admin dashboard
- **Entry points** — 14 route handlers, 9 server actions, 6 Supabase tables, 2 storage buckets, 1 Stripe webhook
- **Trust boundaries** — anonymous → authenticated → service role
- **Not available in the repository** — proxy configuration, deployed RLS state, environment variable values

## Findings

### H-1 — Any authenticated user can read every row of `documents`

| Element | Value |
|---|---|
| Subject | authenticated user (any tenant) |
| Operation | `select` on `documents` with their own session |
| Intended control | Supabase RLS |
| Boundary crossed | tenant → tenant |
| Affected | all tenants' uploaded documents |
| Consequence | a logged-in user can list and download other tenants' documents |

**Evidence:** `supabase/migrations/0007_documents.sql:12` — RLS is enabled, but the policy is `USING (auth.role() = 'authenticated')`, which matches every logged-in user rather than the row owner.

**Smallest fix:** `USING (auth.uid() = owner_id)` with a matching `WITH CHECK`.

---

### H-2 — Stripe webhook accepts unsigned events

| Element | Value |
|---|---|
| Subject | anonymous internet caller |
| Operation | `POST /api/stripe/webhook` |
| Intended control | Stripe signature verification |
| Boundary crossed | anonymous → service role |
| Affected | billing state of any account |
| Consequence | a forged `checkout.session.completed` grants a paid plan without payment |

**Evidence:** `app/api/stripe/webhook/route.ts:9` — the handler parses the body and writes to `subscriptions` with the service-role client. `stripe.webhooks.constructEvent` is never called.

**Smallest fix:** verify the signature against `STRIPE_WEBHOOK_SECRET` before parsing, and reject on failure.

---

### M-1 — `service_role` key reachable from the client bundle

| Element | Value |
|---|---|
| Subject | any visitor |
| Operation | read the shipped JavaScript |
| Intended control | secrets stay server-side |
| Boundary crossed | client bundle → service role |
| Affected | the entire database (this key bypasses RLS) |
| Consequence | full read/write access to every table |

**Evidence:** `lib/supabase/admin.ts:4` reads `SUPABASE_SERVICE_ROLE_KEY`; `app/(app)/dashboard/page.tsx:3` is a `"use client"` component that imports `getAdminClient` from that module.

**Smallest fix:** move the admin client behind a server-only module, import it only from route handlers and server actions, and add `import 'server-only'`.

## needs_validation — no severity assigned

- Whether the deployed Supabase project carries additional RLS policies absent from the migrations. The repository is the only source available.
- Whether a WAF or proxy strips unsigned webhook calls before they reach the handler.
- The values of `NEXT_PUBLIC_*` variables in the deployed environment.

## Coverage statement

This review covers the repository as provided. Domains not exercised — payment reconciliation logic, email deliverability, third-party SDK versions — are out of scope. A review reduces risk; it does not certify the application as safe.
