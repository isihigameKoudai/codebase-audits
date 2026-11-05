# codebase-audits

Codebase reviews and hardening skills for AI-generated (vibe-coded) Next.js + Supabase apps.

> **No calls. Just Pull Requests — and we watch it so you don't have to.**

45% of AI-generated code fails the OWASP Top 10 (Veracode 2025). AI-assisted pull requests introduce security issues at 2.74× the human rate (CodeRabbit). In a study auditing 200 deployed vibe-coded apps, 91% had at least one vulnerability and 65.77% carried Critical/High findings (arXiv:2606.23130).

This repository shows what a real review looks like, and publishes the skills that prevent those problems before they ship.

## Why checklists don't work on AI-generated code

AI coding tools (Lovable, Bolt, Cursor, Replit, v0) are good at generating code that **runs**.
They are unreliable at generating code that **holds a boundary**.

That matters because the expensive bugs aren't in the patterns — they're in the **seams**:

- a Supabase table where RLS is *enabled* but the policy matches the wrong column
- a Next.js server action that checks auth in the UI and not on the server
- a `service_role` key that gets imported into a client component three files away
- a public storage bucket that also holds user-uploaded documents

A fixed checklist can't tell you which seam *your* app crosses. Most public checklists are the same ten items, so they find the same ten things — and stop.

## The method

Four phases. Each produces evidence, not opinion.

### 1. Threat model first

Before reading code for bugs, map what the app actually is:

- **Assets** — user data, payment flows, admin surfaces, uploaded files
- **Entry points** — routes, server actions, RPCs, webhooks, storage policies
- **Trust boundaries** — anonymous → authenticated → service role

Everything downstream is scoped by this. A review without a threat model is a review without a scope.

### 2. Coverage-led hunting

Instead of free-form reading, the review walks a coverage map of **12 domains and ~160 attack classes**, then records which units apply to *your* entry points and which do not.

Two rules govern this phase:

- A candidate must name **the boundary it crosses** and **who is affected**. "A best practice is missing" is not a finding.
- Deployment behaviour, proxy settings and provider configuration are **not in the repository**. They are never assumed — they are listed as `needs validation`.

### 3. Adversarial verification

Every candidate finding is handed to a **separate** pass whose job is to *disprove* it from source.
The reviewer who found it never signs it off. Candidates that can't survive are **dropped**, not softened.

### 4. Structured, severity-ranked output

Confirmed findings are ranked (critical / high / medium / low), each with the exact **file and line**, the **boundary crossed** and **affected subject**, the **smallest fix** that holds the invariant, and a **regression test** where one is meaningful.

## Severity discipline

- Only **verified** findings receive a severity.
- Anything that depends on a fact absent from the repository — deployment config, proxy behaviour, provider settings — is reported as **needs validation**, with the missing fact named. It is never asserted as a vulnerability.

This is why this report contains **fewer findings than a scanner**, and why each one is worth reading.

## What this review does not do

- **No production access.** Source only, read-only. Your live environment is never touched.
- **No calls.** Written and asynchronous end to end.
- **No guarantees.** A review reduces risk; it does not certify an application as safe. Nobody can honestly promise that.

## What's in this repo

| Path | What it is |
| --- | --- |
| [`.agents/skills/vibe-app-security-review/`](.agents/skills/vibe-app-security-review/) | The **review** skill — threat model, coverage-led hunting, adversarial verification, structured findings. |
| [`.agents/skills/vibe-app-hardening-rules/`](.agents/skills/vibe-app-hardening-rules/) | The **prevention** skill — imperative rules for secrets, RLS, authorization and query performance, applied while you code. |
| [`sample-reports/`](sample-reports/) | Anonymized sample codebase reviews — severity-ranked, each with file:line, the vulnerable code, and the fix. Demo projects; every finding is real-world. |
| [`checklists/`](checklists/) | Free checklists you can run yourself today (deliberately limited — see above). |
| [`findings-library/`](findings-library/) | The issues that recur most often across AI-generated codebases, with the smallest fix for each. |

## The skills

Both are **Agent Skills** — the open format (SKILL.md) read by Codex, Cursor, GitHub Copilot, Claude Code and ~70 other tools.

```bash
git clone https://github.com/isihigameKoudai/codebase-audits /tmp/codebase-audits
mkdir -p .agents/skills
cp -r /tmp/codebase-audits/.agents/skills/* .agents/skills/
```

Then ask your agent to *"review this app for security issues"* (review skill) or just let the hardening rules apply while it writes code.

> Earlier versions of this repo shipped these as `.cursorrules` / `.mdc` files. `.cursorrules` is legacy — Cursor's own docs say so — and Cursor-only rules reach one tool. Agent Skills reach the whole ecosystem.

## Services

I review and harden AI-generated Next.js + Supabase apps. I work 100% asynchronously — no calls, no production access, delivered as a pull request you merge yourself.

- **Codebase Audit** — $199 — a severity-ranked report with file:line findings and a fix roadmap.
- **Security Hardening PR** — $899 — a pull request implementing your agreed top-5 fixes.
- **Production Hardening** — $2,400 — PR + refactor + tests + CI.
- **Async Code Guardian** — $1,800/mo — ongoing review and monitoring.

**→ [Fiverr: I will audit your vibe coded Next.js Supabase app for security bugs](https://www.fiverr.com/s/xrkD23x)**

The skills in this repo are the same boundaries I apply in a paid review, published here so you can prevent the problems before they ship.

## Method provenance

The review workflow is adapted from open-source security-audit methodologies:
Cloudflare's `security-audit-skill` (MIT) and Anthropic's `defending-code-reference-harness` (Apache-2.0).
Coverage-led hunting, adversarial verification and structured findings come from those sources — not from a proprietary checklist.

## About me

Senior frontend / full-stack engineer (8+ years — TypeScript, React, Next.js, Supabase).
Based in Japan, working asynchronously worldwide.

---

*License: MIT. The sample reports are illustrative — findings are from demo projects.*
