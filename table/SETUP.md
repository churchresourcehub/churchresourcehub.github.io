# The Table — setup and launch

The Table is a single-page discussion space for clergy, anchored to the Church Resource Hub.
It ships in **preview mode** (browser-only, example posts, nothing shared) until Supabase
credentials are added to `config.js`. Woods does the two account steps below himself; they
take about five minutes together.

## 1. Going live (Woods, ~5 minutes)

1. Create a free project at supabase.com (any project name; region US East).
2. In the SQL editor, run the schema at the bottom of this file.
3. In Authentication → Providers, leave Email enabled (magic links are the default).
4. Copy Project URL and the anon public key from Settings → API into `config.js`, uncommenting both lines.
5. Redeploy. The preview banner disappears and posting requires a sign-in link.

## 2. Domain

The app deploys on Vercel like the other properties. To put it at table.doingchurchtogether.org:
Vercel project → Settings → Domains → add `table.doingchurchtogether.org` (the DNS is already
on Vercel, so it wires itself).

## 3. Before the conference-wide send

- Supabase's built-in email sender is rate-limited to a handful of magic links per hour, which
  is fine for the lunch-table trial and not fine for a conference launch. Before the wide send,
  add a custom SMTP sender in Supabase (Resend's free tier covers 100 emails/day; verify the
  doingchurchtogether.org domain there and paste the SMTP settings into Supabase Auth settings).
- Seed the room: the lunch table posts first, so the first visitor finds real furniture.
- Reply notifications (email when someone answers your thread) are a fast follow, not in v1.
  The clean path is a Supabase database webhook on `replies` insert → a small Vercel function
  → Resend. Say the word when the trial proves out.

## 4. Refreshing the library snapshot

`resources.json` is a snapshot of the hub's catalog (154 resources) used by the attach picker
and the "New in the library" strip. When the hub gains cards, re-extract it from the hub
master's `RESOURCES` array (the session log shows the one-liner) and redeploy.

## Schema (run once in Supabase SQL editor)

```sql
create table threads (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  author_name text not null,
  author_church text,
  area text not null,
  title text not null,
  body text not null,
  resource_title text,
  resource_link text
);
create table replies (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz default now(),
  thread_id uuid references threads(id) on delete cascade,
  author_name text not null,
  author_church text,
  body text not null
);
alter table threads enable row level security;
alter table replies enable row level security;
create policy "read threads" on threads for select using (true);
create policy "read replies" on replies for select using (true);
create policy "post threads" on threads for insert to authenticated with check (true);
create policy "post replies" on replies for insert to authenticated with check (true);
```
