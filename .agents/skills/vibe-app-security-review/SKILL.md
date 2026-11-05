---
name: vibe-app-security-review
description: "Use when reviewing an AI-generated (vibe-coded) Next.js + Supabase app for security before launch. Threat-model-first, boundary-led review with adversarially verified findings."
license: MIT
---

# Vibe-Coded App Security Review (Next.js + Supabase)

A review workflow for apps built primarily by AI coding tools (Lovable, Bolt, Cursor, Replit, v0, Claude Code) on Next.js + Supabase.

**Premise**: AI tools are good at generating code that *runs*, and unreliable at generating code that *holds a boundary*. The expensive bugs are in the seams, not the patterns.

## When to use

- A user wants their vibe-coded / AI-generated Next.js + Supabase app reviewed before launch
- A user asks "is my app safe?" about a codebase they did not write by hand
- A user wants a second opinion on an AI-generated codebase

**Do not use for**: general "is this code good" reviews, single-bug debugging, or compliance audits.

## The one rule that governs everything

> **A finding must name the boundary it crosses and who is affected.**
> "A best practice is missing" is not a finding.

Every candidate must state:

1. The **subject** acting (anonymous visitor / authenticated user / other tenant / service role / webhook caller)
2. The **input or operation** they control
3. The **intended control** that should stop them
4. The **boundary** they cross
5. The **affected subject or resource** (whose data, whose money, whose account)
6. The **observed or owner-observable consequence**

If you cannot name all six, you do not have a finding — you have a hypothesis. Label it as such.

## Phase 1 — Threat model (before reading for bugs)

Write down, from the repository alone:

- **Assets** — user PII, auth sessions, payment/subscription state, admin surfaces, uploaded files, API keys
- **Entry points** — route handlers, server actions, middleware, Supabase RPCs, storage policies, webhooks (Stripe etc.), cron/edge functions
- **Trust boundaries** — anonymous → authenticated → service role; client bundle → server; tenant → tenant
- **Facts you do NOT have** — proxy behaviour, provider settings, environment variable values, deployed RLS state

That last bullet matters. Anything living outside the repository is **not assumed** — it becomes `needs_validation`.

## Phase 2 — Coverage-led hunting

Walk each domain against the entry points from Phase 1.

| Domain | Focus for this stack | Reference |
|---|---|---|
| Data isolation | RLS enabled vs *enforced*; wrong-column policies; `USING (true)`; missing `WITH CHECK`; storage bucket visibility | `references/supabase-rls.md` |
| Auth boundaries | Server actions / route handlers trusting the client; middleware-as-boundary; `getSession()` vs `getUser()`; IDOR | `references/nextjs-auth-boundaries.md` |
| Secrets & bundling | `NEXT_PUBLIC_` misuse; `service_role` reaching the client bundle; `.env` in git history | `references/secrets-and-bundling.md` |
| Injection | AI-generated SQL/RPC strings; `dangerouslySetInnerHTML`; unvalidated input reaching queries | as above |
| Resource exhaustion | Missing pagination; unbounded queries; ReDoS in validators; no rate limit on auth | — |
| Third-party trust | Webhook signature verification; OAuth `state`; redirect allow-lists | as above |

For each domain record **applicable / not applicable / needs validation**, and the entry point it maps to. A domain you skipped is a coverage gap — say so explicitly.

## Phase 3 — Adversarial verification

For every candidate, run a **separate** pass whose only job is to **disprove** it from source.

- The pass that found the issue never signs it off.
- The verifier must actively try to show the boundary *is* enforced somewhere else.
- Candidates that cannot survive are **dropped**, not softened.

Use cheap programmatic gates first (does the code path exist? is the policy actually attached to that table?), then reason.

## Phase 4 — Structured output

For each **confirmed** finding:

- **Severity** — critical / high / medium / low
- **`file:line`**
- **The six elements** from the one rule
- **Smallest effective fix** — the narrowest change that enforces the invariant at the last trusted point
- **Regression test**, where one is meaningful

## Severity

- **critical** — unauthenticated attacker gets code execution, full datastore access, or account takeover
- **high** — a security control is *completely* broken with real impact: auth bypass, cross-tenant read/write, stored XSS affecting other users
- **medium** — a real boundary violation with limited blast radius, rare preconditions, or a narrow set of affected resources
- **low** — disclosure of non-secret internal information, or an effect needing sustained effort for minimal gain
- **informational** — confirmed but minimal impact

> Only **verified** findings receive a severity.
> `needs_validation` items receive **no severity** — they are source-grounded blocked hypotheses, not low-confidence vulnerabilities.

## Anti-patterns — do not do these

1. Reporting a checklist deviation as a vulnerability
2. Defence-in-depth advice with no reachable boundary violation
3. Assuming deployment behaviour (proxies, provider config) absent from the repo
4. Treating a user's own privileges or self-impact as a boundary crossing
5. Reporting a stronger effect than what was actually observed
6. Assigning severity to `needs_validation` items
7. Writing the report before independent verification, or letting prose disagree with the structured findings

## Safety

- **Read-only.** Never modify the target source.
- **No production access.** Never touch a live environment, endpoint, or real user data.
- **No network exploration of deployed systems.**
- Do not install dependencies into the target or let it reach the network.
- If a conclusion needs a fact only the owner can supply, record it as `needs_validation` with the exact missing fact — never guess.

## What this skill is not

A review workflow, not a certification. A clean run does **not** mean the app is safe, and output must never be presented as a guarantee.

## Provenance

Adapted from Cloudflare's `security-audit-skill` (MIT, © Cloudflare) and Anthropic's `defending-code-reference-harness` (Apache-2.0, © Anthropic PBC). Coverage-led hunting, adversarial verification and structured findings come from those sources.
