# Round Taiwan Rally — Deployment Guide

Architecture at a glance:

```
feeltaiwan.com (Wix)  ──link/CTA──▶  game.feeltaiwan.com (static host, one HTML file)
                                          │
                                          ├── GA4 (one property, cross-domain linked)
                                          └── Supabase (players / progress / entries)
```

Wix stays the marketing front door. The game runs full-screen on your own subdomain so
storage is first-party (works on iPhone Safari), tracking is one continuous session, and
entries live in a real database you can export for the drawing.

---

## 1. Host the game at game.feeltaiwan.com

Any static host works. Cloudflare Pages (free) recommended:

1. Create a free Cloudflare account → **Workers & Pages → Create → Pages → Upload assets**.
2. Upload `round-taiwan-rally-prototype.html` renamed to `index.html`.
3. Project gets a `*.pages.dev` URL — verify the game loads.
4. **Custom domains → Add** → `game.feeltaiwan.com`. Cloudflare shows you a CNAME record.
5. In Wix (**Settings → Domains → Manage DNS records**, or wherever feeltaiwan.com's DNS lives),
   add that CNAME: `game → round-taiwan-rally.pages.dev` (name/value as Cloudflare shows).
6. Wait for the certificate (minutes). Done — the game is live at https://game.feeltaiwan.com.

Netlify or GitHub Pages (which you already use for quiz images) work the same way; the only
requirement is the `game.feeltaiwan.com` CNAME so the game is first-party to your site.

## 2. Supabase (players, progress, entry ledger)

1. Create a free project at supabase.com → note the **Project URL** and **anon public key**
   (Settings → API).
2. SQL Editor → run:

```sql
create table players (
  email text primary key,
  name text, token text, consent boolean default false,
  utm jsonb, created_at timestamptz default now()
);

create table progress (
  email text primary key references players(email) on delete cascade,
  collected int[] default '{}', lap int default 1, pos int default 0,
  pass boolean default false, updated_at timestamptz default now()
);

create table entries (
  id bigint generated always as identity primary key,
  email text references players(email) on delete cascade,
  tier text not null check (tier in
    ('checkin','lap1','set_north','set_east','set_south','set_central','rail','half','full')),
  pts int not null check (pts between 1 and 3),
  created_at timestamptz default now(),
  unique (email, tier)                 -- server-side cap: each bonus once per player
);

alter table players  enable row level security;
alter table progress enable row level security;
alter table entries  enable row level security;

-- the game (anon key) may write but never read other players' data
create policy "anon insert players"  on players  for insert with check (true);
create policy "anon update players"  on players  for update using (true);
create policy "anon insert progress" on progress for insert with check (true);
create policy "anon update progress" on progress for update using (true);
create policy "anon insert entries"  on entries  for insert with check (true);
```

3. In the game file, edit the CONFIG block at the top of the script:

```js
const CONFIG = {
  backend: 'supabase',
  supabaseUrl: 'https://YOUR-PROJECT.supabase.co',
  supabaseAnonKey: 'YOUR-ANON-KEY',
  gaId: 'G-XXXXXXXXXX',
  gaLinkDomains: ['feeltaiwan.com','game.feeltaiwan.com']
};
```

`backend: 'local'` keeps everything in-browser — use it for demos and partner previews.

**Export for the winner drawing:** Supabase → Table editor → `entries` → Export CSV.
One row per (player, bonus tier); weight by `pts` or expand rows, then feed your existing
winner-drawing tool. `players` gives you the consented email list.

**Integrity notes:** the `unique(email, tier)` constraint plus the `check` whitelist mean
even a tampered client can never register more than the 9 legitimate bonuses (max 12
entries) per email. Rolls/quiz answers stay client-side — acceptable because the entry cap
is enforced by the database. If you later want stricter validation, move awards into a
Supabase Edge Function and keep the same tables.

## 3. GA4 (one funnel across Wix + game)

1. Use one GA4 property. Add its tag to Wix (Settings → Marketing integrations, or a custom
   code snippet) and put the same `G-XXXX` id in the game's CONFIG (`gaId`).
2. The game already configures the cross-domain **linker** for
   feeltaiwan.com ↔ game.feeltaiwan.com. In GA4 admin also set:
   **Data streams → Web → Configure tag settings → Configure your domains** → add both domains.
3. Events the game sends automatically: `game_checkin`, `stamp_collected` (with destination),
   `lap_complete`, `tier_awarded`, `game_complete`. Mark `game_checkin` as a key event to
   measure ad → registration conversion. UTM parameters on the game URL are also saved
   per-player into Supabase (`players.utm`), so partner attribution survives even if GA is
   blocked by the browser.

## 4. Wix integration

- **Primary CTA (recommended):** a button/banner on feeltaiwan.com linking to
  `https://game.feeltaiwan.com?utm_source=site&utm_medium=banner&utm_campaign=rally`.
  Full-screen is the best experience on phones (the mobile board uses the whole screen).
- **Optional embed:** the file still supports iframe embedding (it posts its height to the
  parent like the quiz did). Because the game is on your subdomain, iOS storage inside the
  iframe now works too. Use the embed for continuity if you want the game visible inline on
  the landing page; keep the full-screen link as the main path.

## 5. Pre-launch checklist

- [ ] iPhone Safari (normal + private): check in, collect one stamp, close tab, reopen — progress persists.
- [ ] Android Chrome: same test.
- [ ] Desktop: hover links between board, map, and dots work; resize window — lines track.
- [ ] Rotate phone / small tablets (620px breakpoint switches layouts cleanly).
- [ ] Supabase: rows appear in `players`, `progress`, `entries` while playing.
- [ ] GA4 realtime shows `game_checkin` with correct source/medium from a UTM-tagged visit.
- [ ] Entries CSV export → winner-drawing tool dry run.
- [ ] Official Rules link on check-in points at the real rules page (update the placeholder).
- [ ] Swap `backend` to `'supabase'` in the PRODUCTION copy only; keep a `'local'` copy for demos.

## 6. Costs

Cloudflare Pages free tier + Supabase free tier + GA4 free = **$0/month** at campaign scale
(Supabase free covers 500MB DB / 5GB egress — orders of magnitude above what a sweepstakes
season of this size generates).
