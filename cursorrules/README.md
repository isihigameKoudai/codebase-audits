# Cursor Rules for Next.js + Supabase

A collection of `.mdc` rule files that stop the most common security and performance mistakes before they land in your code, while you vibe-code with Next.js + Supabase.

## What this is

When you build with Lovable, Bolt, Cursor, or Replit, the model writes code fast — but it doesn't always know which things are dangerous in a real, deployed app. These rules give your coding agent the hard boundaries up front: what never belongs in the browser, when Row Level Security must be on, how to check that the right user owns the row, and how to keep queries from collapsing under load.

Each `.mdc` file is a focused set of imperative rules, each with a short "why", so the agent applies them without needing to understand your whole app.

## Who this is for

You're the target user if:

- You build web apps with Next.js (App Router) + Supabase.
- You write prompts and review code, but you aren't a backend engineer.
- You've seen terms like "RLS" or "service_role" and aren't 100% sure what they do.
- You want your app safe before it's in front of real users.

You don't need to be an engineer to use these. Copy them in and let Cursor (or any agent that reads rule files) enforce them while it generates code.

## Why it exists

Vibe-coded apps ship with the same handful of problems over and over:

- Real API keys committed to the repo or exposed in the browser.
- Tables with Row Level Security off, so anyone with the anon key can read or write everything.
- Server code that checks "is this person logged in?" but never "is this their row?"
- Query loops and missing indexes that fall over with real data.

These rules stop each of those before the code is written, not after. They're the same checks a codebase review applies — baked into the editor so they're enforced from the first prompt.

## Install

Copy the rule files into your project's `.cursor/rules/` directory:

```
mkdir -p .cursor/rules
cp /path/to/cursorrules/*.mdc .cursor/rules/
```

Or, in Cursor, add each `.mdc` file under **Settings → Rules** (they apply project-wide).

That's it. The `alwaysApply: true` frontmatter makes the rules active on every prompt, so you don't need to remember to toggle anything.

## The rules

| File | What it prevents |
| --- | --- |
| `security-secrets.mdc` | Secrets leaking to the browser or the repo |
| `supabase-rls.mdc` | Unprotected tables and bypassed Row Level Security |
| `auth-authorization.mdc` | Handlers that authenticate but don't authorize |
| `performance-queries.mdc` | N+1 queries, missing indexes, unbounded reads |

## About

Built by a developer who reviews and hardens AI-generated Next.js + Supabase apps. These rules are the same boundaries applied in a paid codebase review, published here so you can prevent the problems before they ship.
