# Project Progress Tracker: Supabase backend

Stack: **GitHub Pages** (static frontend) + **Supabase** (Postgres, Auth, RLS). No ORM and no server of your own.

## Files

| File | Purpose |
|---|---|
| `migrations/20261110000000_init.sql` | Schema, roles, Row Level Security, version checks, audit changelog |
| `seed.sql` | The current HTML data: 11 projects, 58 tasks, settings |

## One-time setup

1. **Create the Supabase project.** Choose the region closest to your users. Free projects are paused after a week of inactivity, so use a paid plan if other people rely on this.
2. **Turn on self-service sign-up.** In Authentication → Providers → Email, turn ON "Allow new users to sign up" and turn OFF "Confirm email" (new accounts are active immediately, with no inbox-verification step). Set the **Site URL** to your GitHub Pages address. Anyone who visits the site can now create their own account from the "New here? Create an account" link on the sign-in screen — there is no domain restriction, and no admin step is required before they can log in.

   > **Know the trade-off:** every new sign-up is auto-provisioned as a **viewer** (read-only), but the `projects_select`/`tasks_select` policies let any signed-in viewer read *all* projects and tasks — including the ones from IFC, XW Fintech and other partners. Open sign-up means anyone with any email address can read that data the moment they register. If you later want to restrict this (e.g. to a company email domain, or require admin approval before an account goes active), that's a small change to the sign-up trigger and the `profiles.active` default — ask for it if you need it.
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
