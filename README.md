# Project Progress Tracker: Supabase backend

Stack: **GitHub Pages** (static frontend) + **Supabase** (Postgres, Auth, RLS). No ORM and no server of your own.

## Files

| File | Purpose |
|---|---|
| `migrations/20261110000000_init.sql` | Schema, roles, Row Level Security, version checks, audit changelog |
| `seed.sql` | The current HTML data: 11 projects, 58 tasks, settings |

## One-time setup

1. **Create the Supabase project.** Choose the region closest to your users. Free projects are paused after a week of inactivity, so use a paid plan if other people rely on this.
2. **Lock down sign-ups.** In the Auth settings, turn off "Allow new users to sign up" and set the **Site URL** to your GitHub Pages address. Add people with *Authentication → Users → Invite user*.
3. **Apply the migration.** Either `supabase link --project-ref <ref>` then `supabase db push`, or paste the SQL into the SQL editor.
4. **Load the seed data** by running `seed.sql` once in the SQL editor.
5. **Make yourself admin** after you have signed in once:
   ```sql
   update public.profiles set role = 'admin' where email = 'you@example.com';
   ```
6. **Give editors their projects** (admins can edit everything; viewers are read-only):
   ```sql
   insert into public.project_members (project_id, user_id)
   select p.id, u.id
   from public.projects p, public.profiles u
   where p.short_name = 'IFC SME Scorecard' and u.email = 'editor@example.com';
   update public.profiles set role = 'editor' where email = 'editor@example.com';
   ```

## Roles

| Role | Read | Edit tasks | Projects, users, settings |
|---|---|---|---|
| viewer | everything | no | no |
| editor | everything | only in projects they are a member of | no |
| admin | everything (incl. soft-deleted) | all projects | yes |

Deactivate a user with `update public.profiles set active = false where email = '...'`. They keep their account but can read nothing.

## Using it from the frontend (`supabase-js`)

```js
// Load everything for the UI
const { data: projects } = await sb
  .from('projects')
  .select('*, tasks(*)')
  .is('tasks.deleted_at', null)
  .order('sort_order')
  .order('position', { referencedTable: 'tasks' });

// Edit a task: always send back the version you loaded
const { data, error } = await sb
  .from('tasks')
  .update({ progress: 0.6, status: 'In Progress', version: task.version })
  .eq('id', task.id)
  .select()
  .single();
if (error?.code === 'PT409') { /* someone else changed it: reload and ask the user */ }

// Soft delete (rows are never physically removed)
await sb.from('tasks').update({ deleted_at: new Date().toISOString(), version: task.version }).eq('id', task.id);

// Change history for one task
const { data: history } = await sb
  .from('audit_log').select('*')
  .eq('record_id', task.id).order('at', { ascending: false });
```

Each history row has `user_email`, `at`, `action`, `changed_fields`, and full `before_data` / `after_data`, enough to render "progress 0.2 → 0.6 by X on date".

## Security notes

- The **anon key** in the frontend is public by design. Everything is protected by RLS plus invite-only sign-up.
- **Never** put the `service_role` key in the frontend or the repo.
- `audit_log` is append-only: clients cannot write to it, and a trigger blocks `UPDATE`, `DELETE` and `TRUNCATE` for everyone.
- GitHub Pages sites are public, so keep project data out of the HTML. It should come only from Supabase after login.
- If this is employer data, get IT/security approval before using a third-party cloud database and public hosting.
