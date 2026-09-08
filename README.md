# SOAP

*Read. Reflect. Apply. Pray.*

A daily devotional journal built around the S.O.A.P. Bible study method —
**S**cripture, **O**bservation, **A**pplication, **P**rayer.

This is Phase 1 (Foundation) of the build: Next.js + TypeScript + Tailwind on
the frontend, Supabase (Postgres, Auth, RLS) on the backend, ready to deploy
to Vercel. Later phases add authentication UI, the dashboard, the SOAP
editor, the journal, the calendar, reminders, and settings.

## Stack

- Next.js (App Router) + React + TypeScript
- Tailwind CSS v4
- Supabase (Postgres, Auth, Row Level Security)
- React Hook Form + Zod
- date-fns
- lucide-react

## 1. Install dependencies

```bash
npm install
```

## 2. Create a Supabase project

1. Go to [supabase.com](https://supabase.com) and create a new project.
2. In **Project Settings → API**, copy the **Project URL** and the
   **anon public** key. Never use the `service_role` key here — it must
   never be exposed to the browser.

## 3. Configure environment variables

```bash
cp .env.example .env.local
```

Fill in `.env.local`:

```
NEXT_PUBLIC_SUPABASE_URL=your-project-url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key
```

`.env.local` is git-ignored and must never be committed.

## 4. Apply database migrations

Migrations live in `supabase/migrations/` and are numbered in the order
they must run. Apply them with the Supabase CLI:

```bash
npx supabase login
npx supabase link --project-ref your-project-ref
npx supabase db push
```

Or paste each file's contents into the Supabase Dashboard's SQL Editor, in
order, if you'd rather not use the CLI. Either way, the database should be
fully reproducible from this repo — no manual table creation in the
dashboard.

This creates:

- `profiles` — one row per user, created automatically on signup
- `soap_entries` — one SOAP entry per user per day (`UNIQUE(user_id, entry_date)`)
- `reminder_settings` — one row per user, created automatically on signup
- RLS policies restricting every table to its owning user
- A trigger that creates a `profiles` row and default `reminder_settings`
  row whenever a new `auth.users` row is created

After applying migrations, regenerate the TypeScript types to match your
live schema:

```bash
npx supabase gen types typescript --project-id your-project-ref > src/types/database.ts
```

(A hand-written version matching the migrations already ships in
`src/types/database.ts` so the project type-checks before you've linked a
real project.)

## 5. Run locally

```bash
npm run dev
```

Visit [http://localhost:3000](http://localhost:3000).

## 6. Deploy to Vercel

1. Push this repo to GitHub.
2. Import the repo in [Vercel](https://vercel.com/new).
3. Add the same two environment variables from `.env.local` in the
   Vercel project's **Settings → Environment Variables**, for both
   Production and Preview.
4. Deploy. Vercel runs `npm run build` automatically.

## Project structure

```text
src/
├── app/                  # Next.js App Router pages
├── components/
│   ├── ui/               # Buttons, inputs, cards, modals — generic primitives
│   ├── layout/            # Sidebar, header, mobile nav
│   ├── dashboard/         # Today's SOAP, streak, stats, recent entries
│   ├── soap/              # SOAP form, section, card, detail view
│   ├── journal/           # Journal list, card, search
│   └── calendar/          # Monthly calendar
├── lib/
│   ├── supabase/          # client.ts (browser), server.ts (RSC/actions),
│   │                        middleware.ts (session refresh + route guard)
│   ├── auth/               # auth-related helpers
│   ├── soap/               # SOAP entry CRUD + validation
│   ├── reminders/          # reminder settings + browser notifications
│   └── streaks/            # streak/statistics calculation
├── types/                # Database and domain types
└── proxy.ts              # Next.js 16 middleware (session + route protection)

supabase/
└── migrations/           # Numbered, reproducible SQL migrations
```

## Security notes

- The Supabase `service_role` key is never used in this project — only the
  public URL and anon key, which are safe to expose to the browser.
- Row Level Security is enabled on every user-owned table. The frontend
  never needs to (and never should) filter by `user_id` for security —
  the database enforces ownership regardless of what the client sends.
- `src/proxy.ts` refreshes the auth session on every request and redirects
  unauthenticated users away from `/app`, and authenticated users away from
  `/login`, `/signup`, and `/forgot-password`.

## Roadmap

Not implemented yet, by design (see the full spec for detail): payments/
subscriptions, a Bible API integration, AI features, church/group features,
and the native mobile app. The `profiles` table already carries
`subscription_status` / `subscription_plan` / `subscription_expires_at`
columns so monetization can be layered on later without a schema migration,
but nothing in the MVP reads or enforces them.
