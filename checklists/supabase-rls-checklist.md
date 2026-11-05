# Supabase RLS checklist

A short, deliberately incomplete list. It catches the most common cases — not all of them.

## 1. RLS is enabled (not just policies created)

```sql
SELECT schemaname, tablename, rowsecurity
FROM pg_tables WHERE schemaname = 'public';
```

Every table holding user data should show `rowsecurity = true`.
A policy on a table where RLS is **off** does nothing.

## 2. Policies compare against the right column

```sql
CREATE POLICY "own rows" ON profiles FOR SELECT USING (auth.uid() = user_id);
```

Check what `user_id` actually references. If it is not `auth.users.id`, this policy does not do what it looks like it does.

## 3. No `USING (true)` on user data

```sql
SELECT schemaname, tablename, policyname, qual, with_check
FROM pg_policies WHERE schemaname = 'public';
```

`qual = 'true'` means public. `auth.role() = 'authenticated'` means *any* logged-in user, not the owner.

## 4. Write policies have `WITH CHECK`

`FOR INSERT` / `FOR UPDATE` without `WITH CHECK` lets a user rewrite the owner column.

## 5. Storage buckets match the sensitivity of their content

```sql
SELECT id, name, public FROM storage.buckets;
```

Public bucket = readable by anyone with the URL. Check *what* was uploaded there.

---

**What this checklist cannot tell you:** which boundary *your* app actually crosses, whether the deployed project matches these migrations, or whether a `SECURITY DEFINER` function quietly bypasses all of the above.

That is what the full review is for → see `README.md`.
