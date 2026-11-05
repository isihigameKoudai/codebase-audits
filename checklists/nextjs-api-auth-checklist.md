# Next.js API auth checklist

A short, deliberately incomplete list.

## 1. Every route handler and server action enforces its own authorization

Middleware is not a security boundary, and hiding a button in the UI is not a check.
Server actions are POST endpoints — each one is reachable on its own.

## 2. Identity comes from the verified session

```ts
// ✗ the client chooses who it is
const { userId } = await req.json();

// ✓ the server decides
const { data: { user } } = await supabase.auth.getUser();
```

`getSession()` reads the cookie without verifying it. Use `getUser()` for authorization.

## 3. Ownership is checked, not just authentication

```ts
// ✗ any authenticated user can read any row
await supabase.from('orders').select().eq('id', params.id);

// ✓ scoped to the caller
await supabase.from('orders').select().eq('id', params.id).eq('user_id', user.id);
```

## 4. Webhooks verify their signature

Without verification, anyone can POST a forged event — e.g. a paid subscription.

## 5. Redirects and callbacks use an allow-list

`redirectTo`, `next=`, and OAuth callbacks must be constrained to known origins.

---

**What this checklist cannot tell you:** which of *your* entry points are actually reachable without the intended identity, and what a cross-tenant caller gets back. That requires reading the code against a threat model.
