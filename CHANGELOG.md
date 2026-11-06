# Changelog

## 1.0.0 — 2026-11-05

First public release.

### Added

- **`.agents/skills/vibe-app-security-review/`** — the review workflow. Threat model first, coverage-led hunting, adversarial verification, structured severity-ranked findings. Spec-compliant Agent Skill (`skills-ref validate` passes).
- **`.agents/skills/vibe-app-hardening-rules/`** — the prevention rules for secrets, RLS, authorization and query performance, applied while code is being generated. Spec-compliant Agent Skill.
- **`sample-reports/`** — two anonymized sample codebase reviews, severity-ranked with file:line.
- **`checklists/`** — free, deliberately incomplete checklists.
- **`findings-library/common-findings.md`** — the issues that recur most often across AI-generated codebases.
- **`AGENTS.md`** — repo guidance for coding agents.

### Changed

- Distribution moved from `.cursorrules` / `.mdc` files to **Agent Skills** under `.agents/skills/`. `.cursorrules` is legacy per Cursor's own documentation, and Cursor-only rules reach one tool; Agent Skills are read by Codex, Cursor, GitHub Copilot, Claude Code and a wide range of other clients.
- README rewritten: why checklists fail on AI-generated code, the four-phase method, severity discipline, the no-guarantee policy, and method provenance.

### Notes

- Both skills validate against the Agent Skills specification.
- The review workflow is adapted from Cloudflare's `security-audit-skill` (MIT) and Anthropic's `defending-code-reference-harness` (Apache-2.0).
