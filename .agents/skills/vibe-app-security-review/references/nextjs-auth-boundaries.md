# Next.js auth boundaries — review notes

## The core question

**Where is the last trusted point where authorization is enforced?**

In the App Router that is a **route handler** or a **server action** — never the client, and never middleware alone.

## Patterns to check

### 1. Authorization only in the UI
A button is hidden for non-admins; the server action it calls has no check. The action is still reachable.

### 2. Middleware treated as a security boundary
Middleware runs before the route but does not replace per-route authorization. Matcher patterns are easy to get wrong (`/api/:path*` vs nested routes), and middleware is skipped for some paths.

### 3. `getSession()` used for authorization
`supabase.auth.getSession()` reads the cookie and **does not verify it with the auth server**. Use `getUser()` for authorization decisions.

### 4. Route handlers that trust a client-supplied identity

```ts
// ✗ the client chooses who it is
const { userId } = await req.json();
const { data } = await supabase.from('orders').select().eq('user_id', userId);
```

Identity must come from the verified session, never the payload.

### 5. Missing ownership filter (IDOR)

```ts
// ✗ any authenticated user can read any order
await supabase.from('orders').select().eq('id', params.id);
```

The handler checks *that* the user is logged in, not that the row belongs to them.

### 6. Webhooks without signature verification
Without verification, anyone can POST a forged event — e.g. a paid subscription.

### 7. Server Actions reachable without their intended guard
Server actions are POST endpoints. Each one needs its own authorization; do not rely on the page that renders the form.

### 8. Redirect and callback allow-lists
`redirectTo`, `next=`, and OAuth callbacks must be constrained to known origins.

## What to record

For each entry point: what identity does it require, where is that enforced, and what does an unauthenticated or other-tenant caller actually get?
