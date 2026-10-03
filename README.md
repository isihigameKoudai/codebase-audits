# codebase-audits

Codebase reviews and hardening rules for AI-generated (vibe-coded) Next.js + Supabase apps.

> **No calls. Just Pull Requests — and we watch it so you don't have to.**

45% of AI-generated code fails the OWASP Top 10 (Veracode 2025). AI-assisted pull requests introduce security issues at 2.74× the human rate (CodeRabbit). In a study auditing 200 deployed vibe-coded apps, 91% had at least one vulnerability and 65.77% carried Critical/High findings (arXiv:2606.23130).

This repository shows what a real review looks like, and publishes the rules that prevent those problems before they ship.

## What's here

| Path | What it is |
| --- | --- |
| [`reports/sample-audit.md`](reports/sample-audit.md) | An anonymized sample codebase review — severity-ranked findings, each with file:line, the vulnerable code, and the fix. A demo project; every finding is real-world. |
| [`cursorrules/`](cursorrules/) | A collection of Cursor rule files (`.mdc`) that stop the most common mistakes while you vibe-code — secrets, RLS, authorization, and query performance. |

## The cursor rules

Copy them into your project's `.cursor/rules/` directory and they apply on every prompt:

```bash
mkdir -p .cursor/rules
cp cursorrules/*.mdc .cursor/rules/
```

| File | What it prevents |
| --- | --- |
| `security-secrets.mdc` | Secrets leaking to the browser or the repo |
| `supabase-rls.mdc` | Unprotected tables and bypassed Row Level Security |
| `auth-authorization.mdc` | Handlers that authenticate but don't authorize |
| `performance-queries.mdc` | N+1 queries, missing indexes, unbounded reads |

## About me

I review and harden AI-generated Next.js + Supabase apps. I work 100% asynchronously — no calls, no production access, delivered as a pull request you merge yourself.

- **Codebase Audit** — $199 — a severity-ranked report with file:line findings and a fix roadmap.
- **Security Hardening PR** — $899 — a pull request implementing your agreed top-5 fixes.
- **Production Hardening** — $2,400 — PR + refactor + tests + CI.
- **Async Code Guardian** — $1,800/mo — ongoing review and monitoring.

The rules in this repo are the same boundaries I apply in a paid review, published here so you can prevent the problems before they ship.

---

*License: MIT. The sample report is illustrative — findings are from a demo project.*
