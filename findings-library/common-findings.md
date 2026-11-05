# Findings library

The issues that recur most often across AI-generated Next.js + Supabase codebases, and the smallest fix for each.

## 1. RLS enabled, policy on the wrong column
The policy compares `auth.uid()` to a column that references the table's own `id`, not `auth.users.id`.
**Fix:** compare against the column that actually holds the auth user id, and add `WITH CHECK`.

## 2. `auth.role() = 'authenticated'` mistaken for "owner only"
Any logged-in user can read every row.
**Fix:** `auth.uid() = owner_id`.

## 3. Server action with no authorization
The guard lives in the component that renders the form.
**Fix:** authorize inside the action; treat every action as a public endpoint.

## 4. `getSession()` used for authorization
The cookie is read but not verified.
**Fix:** `getUser()`.

## 5. IDOR in a route handler
Authenticated, but not scoped to the caller.
**Fix:** add the ownership filter to the query, or use RLS as the backstop.

## 6. Unsigned webhook
Anyone can forge a payment event.
**Fix:** verify the signature before parsing; reject on failure.

## 7. `service_role` reachable from the client bundle
A helper that reads the key gets imported into a client component.
**Fix:** server-only module + `import 'server-only'`; trace the import graph, do not just move the file.

## 8. `NEXT_PUBLIC_` prefix on a secret
The value is inlined into the shipped JavaScript.
**Fix:** remove the prefix and read it server-side only; rotate the key.

## 9. `.env` in git history
Deleted later, still readable.
**Fix:** rotate every exposed credential; then clean history.

## 10. Public storage bucket holding private uploads
The bucket is public because the profile images needed to be.
**Fix:** split public assets from user documents into separate buckets with separate policies.

## 11. Unbounded query on a growing table
Works in development, falls over at scale.
**Fix:** paginate, and add the index the query needs.

## 12. No rate limit on auth endpoints
Credential stuffing and OTP brute force are cheap.
**Fix:** rate limit at the edge or in middleware; log failures.

---

Each of these is a *pattern*, not a finding. Whether it applies to your app, and what it lets an attacker reach, depends on your entry points and trust boundaries — which is what a threat-model-driven review establishes.
