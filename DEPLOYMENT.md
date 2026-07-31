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

## 2. Supabase (campaigns, players, progress, entry ledger)

Every run of the game is its own sweepstakes with its own prize pool. The two-day travel
show draws three tickets from the people who were at the show; the annual campaign draws
three more from two months of online play. Those pools must never mix, so everything the
game writes is keyed on `(email, campaign)` rather than on email alone. The same person can
play both — that is two registrations, two passports and two separate stacks of entries.

1. Create a free project at supabase.com → note the **Project URL** and **anon public key**
   (Settings → API).
2. SQL Editor → run:

```sql
-- one row per sweepstakes; the entry window here is what actually guards the prize pool
create table campaigns (
  id text primary key,                 -- matches CONFIG.campaign, e.g. 'travelshow-sf-2027'
  name text,
  prize text,
  opens timestamptz not null,
  closes timestamptz not null
);

create table players (
  email text, campaign text not null references campaigns(id),
  name text, token text, airline text, consent boolean default false,
  utm jsonb, created_at timestamptz default now(),
  primary key (email, campaign)
);

create table progress (
  email text, campaign text not null,
  collected int[] default '{}', lap int default 1, pos int default 0,
  pass boolean default false, updated_at timestamptz default now(),
  primary key (email, campaign),
  foreign key (email, campaign) references players(email, campaign) on delete cascade
);

create table entries (
  id bigint generated always as identity primary key,
  email text not null, campaign text not null,
  tier text not null check (tier in
    ('checkin','lap1','set_north','set_east','set_south','set_central','rail','half','full')),
  pts int not null check (pts between 1 and 3),
  created_at timestamptz default now(),
  unique (email, campaign, tier),      -- each bonus once per player PER CAMPAIGN
  foreign key (email, campaign) references players(email, campaign) on delete cascade
);

-- Is this campaign accepting entries right now? SECURITY DEFINER so the anon key can be
-- judged against the campaigns table without being able to read (or edit) it.
create function campaign_open(cid text) returns boolean
  language sql security definer stable as $$
    select exists (
      select 1 from campaigns c
      where c.id = cid and now() >= c.opens and now() < c.closes);
  $$;
revoke all on function campaign_open(text) from public;
grant execute on function campaign_open(text) to anon;

alter table campaigns enable row level security;   -- no anon policy at all: staff-only table
alter table players   enable row level security;
alter table progress  enable row level security;
alter table entries   enable row level security;

-- the game (anon key) may write but never read other players' data, and may only write
-- into a campaign that exists and is currently inside its entry window
create policy "anon insert players"  on players  for insert with check (campaign_open(campaign));
create policy "anon update players"  on players  for update using (campaign_open(campaign));
create policy "anon insert progress" on progress for insert with check (campaign_open(campaign));
create policy "anon update progress" on progress for update using (campaign_open(campaign));
create policy "anon insert entries"  on entries  for insert with check (campaign_open(campaign));
```

3. Register the campaign before the event, e.g. for a show running 20–21 March 2027:

```sql
insert into campaigns (id, name, prize, opens, closes) values
  ('travelshow-sf-2027', 'Bay Area Travel & Adventure Show',
   '3 round-trip flight tickets to Taiwan',
   '2027-03-20 09:00-07', '2027-03-21 18:00-07');
```

4. In the game file, edit the CONFIG block at the top of the script:

```js
const CONFIG = {
  backend: 'supabase',
  campaign: 'travelshow-sf-2027',       // must match campaigns.id exactly
  campaignName: 'Bay Area Travel & Adventure Show',
  prize: '3 round-trip flight tickets to Taiwan',
  prizeShort: 'a round-trip flight to Taiwan',
  rulesUrl: 'https://feeltaiwan.com/rules/travelshow-sf-2027',
  opens: '2027-03-20T09:00',            // player's own clock; closes the form outside it
  closes: '2027-03-21T18:00',
  accessCode: 'OHBEAR',                 // on-site only; leave '' for online campaigns
  supabaseUrl: 'https://YOUR-PROJECT.supabase.co',
  supabaseAnonKey: 'YOUR-ANON-KEY',
  gaId: 'G-XXXXXXXXXX',
  gaLinkDomains: ['feeltaiwan.com','game.feeltaiwan.com']
};
```

`backend: 'local'` keeps everything in-browser — use it for demos and partner previews.
Leave `opens`/`closes` empty for an always-open build.

**Starting the next campaign:** change `CONFIG.campaign` and the prize/rules/date fields,
insert the matching `campaigns` row, redeploy. Nothing else moves. Saved progress in the
browser is namespaced by campaign id, so a returning player starts the new rally clean
instead of inheriting a half-full passport from a prize that was already drawn.

**Export for the winner drawing:** Supabase → SQL Editor →

```sql
select email, tier, pts, created_at
from entries where campaign = 'travelshow-sf-2027' order by email;
```

One row per (player, bonus tier); weight by `pts` or expand rows, then feed your existing
winner-drawing tool. `select email, name from players where campaign = '…'` gives the
consented email list for that campaign. **Always filter by campaign** — an unfiltered export
mixes two prize pools and invalidates the draw.

**Integrity notes:** `unique(email, campaign, tier)` plus the `check` whitelist mean even a
tampered client can never register more than the 9 legitimate bonuses (max 12 entries) per
email per campaign. `campaign_open()` in the RLS policies is the part that protects a
show-floor prize pool: three tickets for a two-day event is very good odds, so the URL will
get shared, and after `closes` passes the database refuses the write no matter what the
client does. The `accessCode` in CONFIG is visible in the page source — treat it as friction
that keeps the link from spreading casually, not as security. Rolls and quiz answers stay
client-side, which is acceptable because the entry cap is enforced by the database. If you
later want stricter validation, move awards into a Supabase Edge Function and keep the same
tables.

> The date in `campaigns` and the date in `CONFIG` are set in two places on purpose: the
> database one is the enforcement, the CONFIG one is only so the player sees a friendly
> "not open yet" panel instead of a button that fails. Keep them in sync.

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
- [ ] Supabase: rows appear in `players`, `progress`, `entries` while playing, and every one
      of them carries the right `campaign` id.
- [ ] Set `closes` to a minute from now, wait for it to pass, reload: the check-in form is
      replaced by the closed notice and Supabase refuses a forced write.
- [ ] On-site build only: a wrong `accessCode` is rejected and the right one lets you in.
- [ ] GA4 realtime shows `game_checkin` with correct source/medium from a UTM-tagged visit.
- [ ] Entries CSV export → winner-drawing tool dry run.
- [ ] Official Rules link on check-in points at the real rules page (update the placeholder).
- [ ] Swap `backend` to `'supabase'` in the PRODUCTION copy only; keep a `'local'` copy for demos.

## 6. Costs

Cloudflare Pages free tier + Supabase free tier + GA4 free = **$0/month** at campaign scale
(Supabase free covers 500MB DB / 5GB egress — orders of magnitude above what a sweepstakes
season of this size generates).
