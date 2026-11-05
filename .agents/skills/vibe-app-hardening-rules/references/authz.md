# Authentication & Authorization

- Check the session at the top of every Route Handler and Server Action that touches data.
  *Why:* Server-side code has no auth by default; an unauthenticated request can still reach it.

- Distinguish authentication from authorization. Being logged in is not the same as owning the row being edited.
  *Why:* Any logged-in user can pass a different user's id to a handler that only checks "is there a session?"

- Verify the row the user is acting on actually belongs to them before returning or mutating it.
  *Why:* The client sends the id; the server must confirm ownership, not trust it.

- Use `await supabase.auth.getUser()` for authorization checks, not `getSession()`.
  *Why:* `getSession()` just decodes the JWT from a cookie. `getUser()` revalidates the token against the Supabase Auth server.

- Derive the user id from the session and compare it against the row's owner — never rely on a client-sent `user_id` field alone.
  *Why:* Client-supplied identity is user-controlled input.

- In App Router, `await cookies()` before using it in `createServerClient` / `createBrowserClient` from `@supabase/ssr`.
  *Why:* In Next.js 15+ `cookies()` is async; skipping the `await` throws a sync/async error.

- Return `404` (not `403`) when the row exists but belongs to someone else, to avoid leaking which ids exist.
  *Why:* A `403` confirms the row exists; a `404` reveals nothing.

- Apply the same checks in Server Actions as in Route Handlers — both are publicly reachable endpoints.
  *Why:* A Server Action is just another HTTP endpoint and gets no auth for free.

```ts
// app/api/notes/[id]/route.ts
import { createServerClient } from "@supabase/ssr";
import { cookies } from "next/headers";

export async function POST(request: Request) {
  const cookieStore = await cookies();
  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    { cookies: () => cookieStore }
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();
  if (!user) {
    return Response.json({ error: "Unauthorized" }, { status: 401 });
  }

  const { id } = await request.json();

  const { data: row } = await supabase
    .from("notes")
    .select("user_id")
    .eq("id", id)
    .single();

  if (!row || row.user_id !== user.id) {
    return Response.json({ error: "Not found" }, { status: 404 });
  }

  // ... proceed — the caller owns this row
}
```
