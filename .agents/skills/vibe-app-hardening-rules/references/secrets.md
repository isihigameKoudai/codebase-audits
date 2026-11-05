# Secrets handling

- Never put real secrets (API keys, tokens, database passwords) in `NEXT_PUBLIC_*` variables.
  *Why:* `NEXT_PUBLIC_*` values are inlined into the browser bundle at build time. Anyone can read them from DevTools.

- Treat every `NEXT_PUBLIC_*` value as public. Only the Supabase `anon` key may be public.
  *Why:* The anon key is designed for the browser and protected by RLS. Anything else has no place client-side.

- Keep the `service_role` key server-only. Import it only in Route Handlers, Server Actions, or modules marked `server-only`.
  *Why:* `service_role` bypasses Row Level Security entirely. In the browser it grants full database access to anyone.

- Never reference `process.env.SUPABASE_SERVICE_ROLE_KEY` in a client component or any file imported by one.
  *Why:* If it ends up in a client bundle it ships to every visitor.

- Add `import "server-only";` at the top of every module that touches the `service_role` key or server secrets.
  *Why:* Next.js throws at build time if a client component tries to import a module containing it.

- Validate environment variables at startup with zod (or `@t3-oss/env-nextjs`) and fail fast on missing or malformed values.
  *Why:* A missing var silently becomes `undefined`, producing confusing runtime errors instead of a clear startup message.

- Ship a `.env.example` listing every variable the app needs, with placeholder values only — never real secrets.
  *Why:* New contributors and future-you can set up the project without guessing or copy-pasting live secrets.

- Add `.env*` and `.env.local` to `.gitignore`, keeping `.env.example` committed.
  *Why:* A single committed `.env.local` is the most common way secrets leak to a public repo.

- Rotate any secret that was ever committed, even if the commit was reverted.
  *Why:* Reverts and force-pushes don't erase history; the secret may already be in someone's clone or in CI logs.

- Read the `anon` key and URL from `NEXT_PUBLIC_*`, but read `service_role` only from a non-public variable.
  *Why:* This split keeps the unsafe key out of the bundle while keeping the safe keys available to client and server.

```ts
// env.ts — validate once at startup, fail loudly
import { z } from "zod";

const envSchema = z.object({
  NEXT_PUBLIC_SUPABASE_URL: z.string().url(),
  NEXT_PUBLIC_SUPABASE_ANON_KEY: z.string().min(1),
  SUPABASE_SERVICE_ROLE_KEY: z.string().min(1),
});

export const env = envSchema.parse(process.env);
```
