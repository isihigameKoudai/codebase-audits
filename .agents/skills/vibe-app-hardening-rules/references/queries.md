# Query performance

- Avoid N+1 queries: never loop over rows issuing one query per row.
  *Why:* Each query is a round trip; 100 rows become 101 queries and page load slows linearly.

- Fetch related rows in a single query with `.in()` on the parent ids instead of a per-row `select`.
  *Why:* One round trip replaces many.

- Prefer a single RPC / database function for complex multi-table reads over several round trips from the client.
  *Why:* The database can join in one pass; orchestrating joins in application code is slower.

- Add indexes on columns used in `where` and `order by`, and on foreign keys.
  *Why:* Without an index, Postgres scans the whole table on every filtered query.

- Reuse a module-level Supabase client for server-only `service_role` operations instead of creating a new one per request.
  *Why:* Each `createClient` opens a new connection pool; per-request creation leaks connections and slows response times.

- Keep `createServerClient` request-scoped in Route Handlers and Server Actions (it must read cookies), but share one client for admin-only work.
  *Why:* The request-scoped client needs the request's cookies; the admin client has no request context and should be shared.

- Paginate lists with `.range(from, to)` — never `select()` an entire table into memory.
  *Why:* Unbounded selects return every row; large tables time out and waste memory.

- Select only the columns you need instead of `*` on wide tables.
  *Why:* Less data over the wire means faster queries and smaller payloads.

- Add `count: "exact"` only when you truly need a total row count.
  *Why:* Exact counts add a full `COUNT(*)` on every request, which is expensive on large tables.

```ts
// lib/supabase-admin.ts — shared server-only client, created once
import { createClient } from "@supabase/supabase-js";

export const supabaseAdmin = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!,
  { auth: { persistSession: false } }
);
```

```sql
-- 0002_indexes.sql
create index notes_user_id_idx on notes (user_id);
create index notes_created_at_idx on notes (created_at desc);
```
