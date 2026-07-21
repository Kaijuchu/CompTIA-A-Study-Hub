# Cross-device sync setup (~5 minutes, free)

This connects A+ Study Hub to a free Supabase project so your progress follows you
between your PC, laptop and phone. Anyone else using your link can make their own
account too — everyone's data stays private to them.

---

## 1. Create the project

1. Go to **https://supabase.com** and sign up (free, no card).
2. Click **New project**.
   - **Name:** `aplus-study-hub`
   - **Database password:** generate one and save it somewhere. You won't need it for
     the app, but you'll want it if you ever administer the database directly.
   - **Region:** pick the one closest to you.
3. Wait ~2 minutes while it provisions.

## 2. Create the table

1. In the left sidebar click **SQL Editor** → **New query**.
2. Paste this in and click **Run**:

```sql
-- one row per user, holding their whole progress blob
create table if not exists public.progress (
  id         uuid primary key references auth.users on delete cascade,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

-- Row Level Security: each user can only ever touch their own row
alter table public.progress enable row level security;

create policy "read own progress"   on public.progress
  for select using (auth.uid() = id);
create policy "insert own progress" on public.progress
  for insert with check (auth.uid() = id);
create policy "update own progress" on public.progress
  for update using (auth.uid() = id) with check (auth.uid() = id);
```

You should see **Success. No rows returned.**

> **Why this matters:** the app ships with a public "anon key" — that's by design.
> Row Level Security is what actually protects the data: without these policies,
> anyone could read everyone's rows. With them, the database refuses any request
> for a row that isn't yours. This is the single most important step.

## 3. (Optional) Skip email confirmation

By default Supabase emails a confirmation link before a new account can sign in.
For a personal study app that's usually just friction:

- **Authentication** → **Sign In / Providers** → **Email**
- Turn **Confirm email** off → **Save**

Leave it on if you'd rather have the extra check.

## 4. Copy your two values

- Left sidebar → **Project Settings** (gear) → **API keys** (or **Data API**).
- Copy the **Project URL** — looks like `https://abcdefgh.supabase.co`
- Copy the **anon** / **public** key — a long string starting `eyJ...`

> Copy the **anon public** key, never the **service_role** key. The service_role key
> bypasses Row Level Security entirely and must never go in a web page.

## 4b. (Optional) Enable "Continue with Google"

Nicer than passwords, but it needs credentials from Google. Roughly 10 minutes.

### First, tell Supabase which addresses are allowed back

**Authentication** → **URL Configuration**:

- **Site URL:** `https://kaijuchu.github.io`
- **Redirect URLs** — add both, one per line:
  ```
  https://kaijuchu.github.io/**
  http://localhost:8765/**
  ```

Without the localhost entry, Google sign-in works on the live site but not on your
local copy.

### Create the Google credentials

1. Go to **https://console.cloud.google.com** and sign in.
2. Top-left project dropdown → **New Project** → name it `aplus-study-hub` → **Create**.
3. Left menu → **APIs & Services** → **OAuth consent screen**:
   - User type: **External** → **Create**
   - App name: `A+ Study Hub`, plus your email in the two support-email fields
   - **Save and continue** through Scopes and Test users (nothing to add)
   - On the summary page, click **Publish app** → **Confirm**
     (while it's in "Testing", only accounts you list can sign in)
4. Left menu → **Credentials** → **Create credentials** → **OAuth client ID**:
   - Application type: **Web application**
   - Name: `A+ Study Hub web`
   - Under **Authorised redirect URIs** → **Add URI**, paste your Supabase callback:
     ```
     https://YOUR-PROJECT-REF.supabase.co/auth/v1/callback
     ```
     (Supabase shows this exact URL on its Google provider page — copy it from there.)
   - **Create**, then copy the **Client ID** and **Client secret**.

### Paste them into Supabase

- **Authentication** → **Sign In / Providers** → **Google**
- Toggle **Enable Sign in with Google**
- Paste the **Client ID** and **Client secret** → **Save**

That's it — the **Continue with Google** button in the app now works. If it reports
"provider is not enabled", the toggle didn't save; if Google shows a
`redirect_uri_mismatch` error, the callback URI in step 4 doesn't exactly match the
one Supabase displays.

## 5. Connect the app

1. Open the app → **Settings** → **Account & sync**.
2. Paste the Project URL and anon key → **Save connection**.
3. Enter an email and password → **Create account**.

Done. Your progress now uploads automatically a few seconds after each change, and
pulls down whenever you open the app or hit **Sync now**.

---

## How merging works

If you study on two devices, nothing is lost:

- **Question stats** — counters take the higher value on each side, so answering
  10 questions on your laptop and 5 on your phone gives you 15.
- **AI-generated questions** — combined from both devices, duplicates removed.
- **Streak / daily goal** — the later date wins.
- **Settings** (theme, goals, exam date) — whichever device you changed most recently.
- **Unfinished quiz** — the most recently touched one, and a quiz open on *this*
  device is never overwritten mid-session.

## Notes

- **API keys:** you chose to sync these. Settings → Account & sync has a per-device
  toggle if you ever want to turn that off. Use a password you don't reuse elsewhere.
- **Free tier limits:** Supabase pauses projects after ~1 week of no activity — just
  open the dashboard to wake it. Storage is far beyond what this app needs.
- **Sharing your link:** other people get their own accounts and their own private
  data automatically. Your Supabase project just hosts them.
- **Offline:** everything still works signed out or offline; the app syncs next time
  it can reach the network.
