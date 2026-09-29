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
- Notifications are built (see section 5). They go live once Woods finishes the steps there.

## 4. Refreshing the library snapshot

`resources.json` is a snapshot of the hub's catalog (154 resources) used by the attach picker
and the "New in the library" strip. When the hub gains cards, re-extract it from the hub
master's `RESOURCES` array (the session log shows the one-liner) and redeploy.

## 5. Notifications (email and phone alerts)

Signed-in people get a **Notifications** button. They choose how to be reached (email, and/or
an alert on each phone or computer they turn it on for) and what to hear about: replies to their
posts, replies in conversations they joined, every reply, and new conversations (all, or only in
chosen ministry areas). Nobody is ever notified about their own posts. The page side is live;
the sending side needs these one-time steps, all in dashboards Claude cannot reach.

**A. Database (2 minutes).** Supabase → SQL Editor → paste and run `notifications.sql`.
It stamps new posts and replies with the author's account, and adds the settings and device tables.
It also switches on **guest posting** (name and church, no email, no notifications; posts show a
Guest label and are length-capped) and makes signed-in posts carry the poster's own account.
Until it runs, choosing "post as a guest" ends with a note that guest posting is not on yet.

**B. Email sender (10 minutes).** Create a free Resend account (resend.com). Domains → Add
`doingchurchtogether.org`. Resend shows three or four DNS records; the domain's DNS is on Vercel,
so Claude can add them for you through the Vercel connection if you paste them in. Once the domain
shows Verified, API Keys → Create (sending access) and keep the key for step C.

**C. The sender function (10 minutes).** Supabase → Edge Functions → Deploy a new function →
"Via editor", name it `table-notify`, paste `supabase/functions/table-notify/index.ts`, and in
its settings turn **off** "Verify JWT". Then Edge Functions → Secrets, add:

| Name | Value |
|---|---|
| RESEND_API_KEY | the key from step B |
| FROM_EMAIL | `The Table <table@doingchurchtogether.org>` |
| SITE_URL | `https://churchresourcehub.github.io/table/` (change when The Table moves to its own domain) |
| VAPID_PUBLIC_KEY, VAPID_PRIVATE_KEY, VAPID_SUBJECT | copy the three lines from `.secrets/vapid-keys.txt` (local only, never committed) |
| WEBHOOK_SECRET | any long random phrase; reuse it in step D |

**D. Triggers (5 minutes).** Supabase → Database → Webhooks → Create, twice:
one on table `threads`, one on `replies`, event **Insert** only, type **Supabase Edge Functions**,
function `table-notify`, method POST, and add an HTTP header `x-webhook-secret` with the phrase
from step C.

**E. Test.** Sign in, open Notifications, tick "Every new conversation" and email, save; from a
second email address start a conversation. The email should arrive within a minute. For phone
alerts on iPhone: open The Table in Safari, Share → Add to Home Screen, open it from the home
screen, then turn on "On this phone or computer." Android and desktop browsers need no install.

**Limits.** Resend's free tier sends 100 emails a day and 3,000 a month. That covers the trial;
a busy Table with many "every reply" subscribers would need the $20 plan. Resend can also serve
as the Supabase magic-link sender (section 3), which removes that bottleneck at the same time.

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
