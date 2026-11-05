# AGENTS.md

Guidance for AI coding agents working in this repository.

## What this repo is

Sample reports, free checklists, and two Agent Skills for security-reviewing and hardening AI-generated (vibe-coded) Next.js + Supabase apps.

## Rules for agents working here

- **Anonymize.** Never include real client code, names, domains, or credentials in `sample-reports/`. Replace identifiers with `tenant-a`, `example.com`, `[REDACTED]`.
- **Every finding cites `file:line` and names the boundary it crosses.** Do not add "best practice" advice without a reachable violation.
- **Do not add a finding you cannot verify from the source in front of you.** Use `needs_validation` instead, and name the missing fact.
- **Keep `checklists/` deliberately incomplete.** The full method lives in `.agents/skills/vibe-app-security-review/`.
- **No guarantees.** Never write "certified secure", "guaranteed safe", or any outcome promise. See `README.md`.

## Skills

- `.agents/skills/vibe-app-security-review/` — the **review** workflow (audit an existing app)
- `.agents/skills/vibe-app-hardening-rules/` — the **prevention** rules (apply while writing code)

`.agents/skills/` is the cross-tool path: Codex, Cursor and GitHub Copilot all discover skills there.
