---
name: vibe-app-hardening-rules
description: "Use when writing or generating Next.js + Supabase app code, to enforce security and performance boundaries up front. Imperative rules for secrets, RLS, authorization and queries."
license: MIT
---

# Next.js + Supabase Hardening Rules

Rules to apply **while the code is being written**, not after. Each rule carries a short *why*, so it can be applied without understanding the whole app.

**Who this is for** — people building with Next.js (App Router) + Supabase using an AI coding tool (Lovable, Bolt, Cursor, Replit, v0) who write prompts and review code but are not backend engineers.

**Why it exists** — vibe-coded apps ship with the same handful of problems over and over:

- real API keys committed to the repo or exposed in the browser
- tables with Row Level Security off, so anyone holding the anon key can read or write everything
- server code that checks *"is this person logged in?"* but never *"is this their row?"*
- query loops and missing indexes that fall over with real data

These rules stop each of those **before the code is written**.

## Rules

| Reference | Read it when you are |
|---|---|
| `references/secrets.md` | touching env vars, API keys, `.env`, or anything client-side |
| `references/rls.md` | writing SQL migrations, creating tables, or adding policies |
| `references/authz.md` | writing a route handler, server action, or middleware |
| `references/queries.md` | writing a Supabase query or adding a list view |

## How to apply

1. Before generating code that touches data, read the matching reference.
2. Follow the rules literally — they are **boundaries, not preferences**.
3. If a rule cannot be satisfied, say so explicitly rather than silently working around it.
4. Check the finished code against the rules again. A rule applied in one file and forgotten in the next is the normal failure mode.

## Relationship to review

These rules **prevent** problems. They do not prove the app is safe — a codebase review is what **verifies** it. See the repository README.
