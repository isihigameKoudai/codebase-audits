# Supabase RLS — review notes

## The failure mode that matters

**RLS "enabled" is not RLS "enforced".**

### 1. RLS never enabled on the table

```sql
SELECT schemaname, tablename, rowsecurity
FROM pg_tables WHERE schemaname = 'public';
```

Any table holding user data with `rowsecurity = false` is fully readable with the public anon key.

### 2. Policy on the wrong column

```sql
CREATE POLICY "own rows" ON profiles FOR SELECT USING (auth.uid() = user_id);
```

If `profiles.user_id` is a foreign key to `profiles.id` — not to `auth.users.id` — this matches nothing useful, or matches the wrong user. **Verify what the compared column actually references.** This is the single most common AI-generated RLS bug.

### 3. `USING (true)`, or `auth.role() = 'authenticated'`

The first is explicitly public. The second lets *any* logged-in user read *all* rows — often mistaken for "only the owner".

### 4. Write policies without `WITH CHECK`

`FOR INSERT` / `FOR UPDATE` policies need `WITH CHECK`. Without it a user can rewrite a row's `user_id` to someone else's.

### 5. `service_role` reachable from the client

`service_role` bypasses RLS entirely. If it reaches the browser, RLS is decorative. See `secrets-and-bundling.md`.

### 6. Storage buckets

```sql
SELECT id, name, public FROM storage.buckets;
```

A `public` bucket is readable by anyone holding the URL. Check *what* was uploaded into it, not just that it is public.

### 7. `SECURITY DEFINER` functions

These run with the definer's privileges and can bypass RLS. Read the body for missing authorization.

## What to record

- Per table with user data: RLS enabled? which policy? which column? what does that column reference?
- Per bucket: public? what content lands there?
- Anything requiring deployed project settings → `needs_validation`
