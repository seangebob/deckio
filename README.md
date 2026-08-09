 # Packt

Turn a community into a trading card set. Make a card about yourself, add it to
your event's shared binder, and earn one sealed pack — open it to pull one random
card of someone else in the room.
---

## About

Every hackathon starts the same way: 100+ people who will spend a weekend three feet
apart and leave knowing four of them. Icebreakers don't fix it. A Slack channel of
introductions is a wall of text nobody scrolls, and a name tag tells you a name.

Trading cards solved this decades ago. You make one, you put it in the pool, and the
only way to see anyone else's is to open a pack. **The scarcity is the mechanic.** You
don't browse a directory of 150 strangers and message nobody — you get *one*, at
random, and that one person becomes someone you actually go find in the room.

Packt is that loop, shipped:

1. **Make your card.** One screen — a photo, two prompts from a catalogue of eight, a
   fun fact, a link. The card assembles live beside the form as you type.
2. **Contributing earns you a pack.** Not a feed, not a directory. One sealed pack per
   card you add (up to five per set).
3. **Open it.** A foil pack rattles, tears, and falls away on a dark stage, revealing
   one random card from the pool — never your own, never one you already hold.
4. **The binder.** Nine-pocket pages with ring holes punched down the margin. Empty
   pockets stay visible, so the set visibly wants more people in it.

The result is a public binder: a collage of the people around you during the event.

Every card is public. **There is no private binder** — "your cards" is a filter on the
same shared page (`?collector=me`), which is also how you view anyone else's
(`?collector=<user-id>`). That's not a promise in the copy; it's the route shape.

> **Live demo:** deploy your own in ~10 minutes with the [Installation](#installation)
> steps below, or drop your Vercel URL here.

---

## Features

| | |
| --- | --- |
|**Live card composer** | Photo, two of eight prompts, a fun fact and a link — with the finished card rendering next to the form as you type. |
|**The pack-opening ceremony** | The one dark surface in the product. A foil pack rattles under tension, the seam flashes, the ripped crown sails off the top of the frame and the body drops out the bottom, and the card arrives. |
|**Nine-pocket binder** | Real binder geometry — punched rings, page turns from the edge of the sheet, and empty pockets left in place so a thin set reads as an invitation. |
|**Three views, one route** | The whole set, your lens, or any other member's — same URL, same component, same query. |
|**Bookshelf of sets** | Every binder stands as a spine on a plank; hover pulls one forward. Organisers create new event binders in-app. |
|**Card as an object** | A keylined art window in a coloured mat, a pointer-driven foil sheen, a 3D tilt, a stamped serial and set code, and a printed back it flips to. |
|**Realtime binder** | New contributions land in open binders without a refresh, with a polling fallback for large rooms. |
|**Rules as database invariants** | Six collection rules enforced by constraints, triggers and row locks — not by frontend checks. |
|**Phone-first** | The whole loop is designed for a photo taken and uploaded on the device it was shot on, over event wifi. |
|**Accessible** | Zero WCAG 2.1 A/AA violations from `axe-core` across all five screens; every animation has a `prefers-reduced-motion` path. |

---

## The heart of it

Four pieces of the codebase that carry the most weight.

### 1. The draw is atomic, and cheating is unrepresentable

`pulls` and `pack_grants` have **no INSERT policy at all**. Nothing holding the
publishable key can write them — the only path is a `security definer` function. A
client cannot fabricate a pull, pull its own card, pull a duplicate, or open a pack it
never earned, regardless of what the frontend does.

[`supabase/migrations/0003_functions.sql`](supabase/migrations/0003_functions.sql)

```sql
update pack_grants set consumed_at = now()
 where id = (
   select id from pack_grants
    where user_id = v_user and pack_id = p_pack_id and consumed_at is null
    order by created_at
      for update skip locked          -- a double-click can't spend one pack twice
    limit 1
 )
returning id into v_grant;

select c.* into v_card
  from cards c
 where c.pack_id  = p_pack_id
   and c.owner_id <> v_user           -- never your own card
   and not exists (                   -- never one you already hold
     select 1 from pulls p
      where p.user_id = v_user and p.card_id = c.id
   )
 order by random() limit 1;
```

Two decisions we'd make again:

- **`FOR UPDATE SKIP LOCKED`** means two concurrent calls — a double-click, two tabs —
  cannot claim the same grant. Exactly one wins.
- **Ineligible cards are excluded in the `WHERE`** rather than drawn-and-retried. A
  duplicate isn't unlikely, it's *impossible to represent*. No retry cap to tune, and
  no unlucky-streak failure mode in a small pool.

The full set of invariants:

| Rule | Where it lives |
| --- | --- |
| Contributing earns exactly one pack | `grant_pack_on_contribution` trigger, same transaction as the insert |
| At most five cards per member per pack | `enforce_cards_per_pack_limit` trigger, serialised by an advisory lock |
| One pack consumed per opening | `UPDATE … FOR UPDATE SKIP LOCKED` in `open_pack` |
| Never your own card, never a duplicate | excluded in `open_pack`'s `WHERE` |
| Duplicate backstop | `unique (user_id, card_id)` on `pulls` |
| A dry pool refunds the pack | `open_pack` resets `consumed_at` before raising `pool_exhausted` |
| Serials don't collide | `assign_card_serial` bumps a counter on `packs` under a row lock |

### 2. One query serves all three binder views

"Mine" is not a separate resource. A collector *has* a card if they contributed it or
pulled it, and that single definition powers the public set, your lens and anyone
else's.

[`supabase/migrations/0003_functions.sql`](supabase/migrations/0003_functions.sql) ·
[`app/packs/[slug]/page.tsx`](app/packs/[slug]/page.tsx)

```sql
create function binder_cards(p_pack_id uuid, p_collector uuid default null)
returns setof cards language sql stable as $$
  select c.* from cards c
   where c.pack_id = p_pack_id
     and (
       p_collector is null                                  -- the whole set
       or c.owner_id = p_collector                           -- they made it
       or exists (select 1 from pulls p                      -- or they pulled it
                   where p.card_id = c.id and p.user_id = p_collector)
     )
   order by c.serial;
$$;
```

Left as `SECURITY INVOKER` on purpose, so row-level security still applies.

### 3. Free-tier limits solved in the browser, not the bill

Supabase's free plan gives 1 GB of storage and keeps server-side image resizing behind
Pro. Straight-from-the-phone photos are 3–5 MB, which would cap an entire event at
roughly **250 cards**. So every photo is downscaled *client-side* into a ~150 KB WebP
for the card and a ~25 KB thumbnail for the grid — the same 1 GB now holds about
**5,700 cards**.

[`lib/images.ts`](lib/images.ts)

```ts
// `imageOrientation: "from-image"` is required: without it, phone photos carrying
// EXIF rotation are drawn sideways onto every card.
bitmap = await createImageBitmap(file, { imageOrientation: "from-image" });

const full  = await toBlob(drawScaled(bitmap, FULL_EDGE).canvas);   // 1200px, card
const thumb = await toBlob(drawScaled(bitmap, THUMB_EDGE).canvas);  //  400px, grid
```

iPhone HEIC won't decode in some browsers at all, so that path throws a typed error
that asks for a JPEG instead of failing silently. Uploads land under `{uid}/{uuid}/`,
which is exactly what the storage RLS policy checks — members can't write into each
other's folders.

### 4. The pack that actually comes apart

The wrapper is **two full-size layers of the same foil in the same box**, clipped along
a ragged seam, with the body's cut sitting slightly above the crown's so they always
overlap — a sealed pack shows no seam at all. Tearing runs four beats on one clock: a
damped rattle, a flash where the seam gives, the crown lifting out of frame, and the
body falling away under gravity. Each piece holds full opacity until it's most of the
way gone, so nothing dissolves mid-stage.

[`app/globals.css`](app/globals.css) ·
[`components/pack/PackOpening.tsx`](components/pack/PackOpening.tsx)

```css
@keyframes pack-crown-lift {
  0%   { transform: translate3d(0, 0, 0) rotate(0deg); opacity: 1; }
  14%  { transform: translate3d(-1%, -6%, 0) rotate(-2.4deg); }  /* the catch */
  70%  { opacity: 1; }                                          /* fade only at the end */
  100% { transform: translate3d(3%, -72vh, 0) rotate(9deg); opacity: 0; }
}
```

Meanwhile the request is in flight. The tear starts immediately so the click feels
answered, but the reveal is held until the response lands — racing them means a slow
network shows a torn-open pack with nothing inside.

```ts
const [response] = await Promise.all([
  fetch(`/api/packs/${slug}/open`, { method: "POST" }),
  wait(TEAR_MS),
]);
```

---

## Installation

### Prerequisites

- **Node.js 20+** and npm
- A free **[Supabase](https://supabase.com)** project
- A **Google OAuth** client (sign-in is Google-only)

No Docker required — migrations push straight to the hosted project.

### 1. Clone and install

```bash
git clone <your-repo-url> packt
cd packt
npm install
```

### 2. Configure Supabase auth

In your Supabase dashboard:

- **Auth → Sign In / Providers → Google** — enable it and paste your Google OAuth
  client ID and secret.
- **Auth → URL Configuration → Redirect URLs** — add `http://localhost:3000/auth/callback`
  and `https://<your-app>/auth/callback`.

In the Google Cloud console, add those same two URLs as authorised redirect URIs, plus
`https://<project-ref>.supabase.co/auth/v1/callback`.

### 3. Environment

Create `.env.local` in the project root:

```bash
# Project Settings → API Keys
NEXT_PUBLIC_SUPABASE_URL=https://<project-ref>.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=<publishable-key>

# Optional — sets the OAuth redirect origin in production
NEXT_PUBLIC_SITE_URL=https://<your-app>

# Optional — set to 0 to fall back to 5s polling instead of Realtime
NEXT_PUBLIC_REALTIME=1
```

**No service-role key is needed anywhere.** Every write goes through RLS or a
`security definer` function.

### 4. Database

```bash
npx supabase login
npx supabase link --project-ref <your-project-ref>
npx supabase db push          # applies supabase/migrations/*.sql
```

Grant yourself organiser rights so you can create event binders in-app — insert your
Google email, lowercase, into `admin_emails`:

```sql
insert into admin_emails (email) values ('you@gmail.com');
```

Optionally seed a dozen staff cards so the first pack anyone opens isn't empty: paste
[`supabase/seed.sql`](supabase/seed.sql) into the SQL editor.

Whenever the schema changes, regenerate the typed client:

```bash
npx supabase gen types typescript --linked > lib/database.types.ts
```

### 5. Run

```bash
npm run dev     # http://localhost:3000
npm run build   # production build
npm run lint    # eslint
```

### 6. Deploy

Import the repo on [Vercel](https://vercel.com), add the same environment variables,
and deploy. Add the deployed `/auth/callback` URL to both Supabase and Google, then
point `NEXT_PUBLIC_SITE_URL` at it.

> Vercel Hobby can't connect to a repo owned by a GitHub organisation — use a personal
> repo or a Vercel team.

---

## Using Packt

**On the web or on your phone** — Packt is a responsive web app, so there's nothing to
install for attendees. Share the binder link (or a QR code to it) and it works the same
in a phone browser as on a laptop.

**As an attendee**

1. Open the event's binder link and **sign in with Google**.
2. Hit **Add your card** → upload a photo (shooting it on your phone right there is the
   intended path), answer two prompts, add a fun fact and a link. Watch it assemble
   beside the form.
3. Submit. You've just earned a sealed pack.
4. Hit **Open your pack** and tear it. You get one random card from the pool.
5. Go find that person. Come back to the binder to see your pockets fill in.

**As an organiser**

1. Add your Google email to `admin_emails` (see step 4 above).
2. Sign in, then **All Binders → New binder** — name it, pick an accent colour and a set
   code. Attendees can start contributing immediately.
3. Share `https://<your-app>/packs/<slug>`. The `/packs/<slug>/about` page explains the
   loop to first-timers.

---

## Tech stack

| Layer | Choice |
| --- | --- |
| **Framework** | Next.js 16 (App Router, Server Components, Route Handlers) |
| **UI** | React 19 · TypeScript 5 · Tailwind CSS v4 |
| **Database** | Supabase Postgres — triggers, `security definer` functions, row-level security |
| **Auth** | Supabase Auth, Google OAuth, cookie sessions via `@supabase/ssr` |
| **Storage** | Supabase Storage, per-user prefixes enforced by RLS |
| **Realtime** | Supabase Realtime, with a polling fallback |
| **Validation** | Zod schemas shared by the form and the route handlers |
| **Hosting** | Vercel — card images served through `next/image` so repeat views hit the CDN, not Supabase egress |
| **Animation** | Hand-written CSS keyframes and 3D transforms — no animation library |

Architecturally: **reads go through Server Components; only the two mutations are Route
Handlers.** Business logic lives in framework-free modules under `lib/services/` that
take a Supabase client and plain arguments, so it's testable without a request.

There is no separate backend service and no AI image step — see
[`docs/decisions.md`](docs/decisions.md) for the reasoning.

---

## Project structure

```
app/
  page.tsx                      landing — three live cards as the hero
  packs/page.tsx                every set, as spines on a bookshelf
  packs/[slug]/page.tsx         THE BINDER — ?collector=me|<uuid>, 9 pockets a page
  packs/[slug]/about/           per-set welcome: how the loop works
  packs/[slug]/contribute/      one screen with a live card preview
  packs/[slug]/open/            the pack-opening ceremony (the one dark surface)
  admin/packs/new/              organiser-only: create an event binder
  api/packs/[slug]/cards        POST — create + contribute in one transaction
  api/packs/[slug]/open         POST — open_pack RPC
  auth/callback/                OAuth code exchange
  globals.css                   the design system: card, binder, bookshelf, foil pack

components/
  card/                         the layered card: frame, foil sheen, 3D tilt, flip
  binder/                       nine-pocket grid, collector filter, realtime hook
  bookshelf/                    the shelf of binders
  create/                       contribute form with live preview
  pack/                         the sealed pack and its tear choreography

lib/
  services/                     cards · packs · draw · binder · profiles (framework-free)
  supabase/                     browser, server and proxy clients
  images.ts                     browser-side downscale + upload
  schemas.ts                    Zod contracts shared by form and API
  prompts.ts                    the eight-question catalogue
  database.types.ts             generated from the live schema

supabase/
  migrations/                   schema, triggers, functions, RLS, storage, realtime
  seed.sql                      a dozen staff cards
  tests/invariants.sql          the rules, as assertions

scripts/
  generate-seed-images.mjs      regenerates public/seed/*.png
```

---

## Running a live event

- Free Supabase projects **pause after a week idle** — wake yours the day before.
- After a test upload, check the Storage dashboard: derivatives should be ~150 KB and
  ~25 KB. Megabytes means the browser-side downscale fell through.
- Realtime is free up to 200 peak connections and 2M messages/month. Messages are
  counted per listening client — 150 viewers × 200 contributions is ~30,000, well
  inside quota. **Connections** are the real ceiling: set `NEXT_PUBLIC_REALTIME=0` if
  you expect more than ~200 simultaneous viewers.
- Seed a few staff cards before doors open so the first pack of the event isn't empty.

## Roadmap

- **Rarity that means something.** The column exists and everything is `common` today —
  earliest contributors as holos, that sort of thing.
- **Trading**, with two-sided consent. The constraint work is already the right shape.
- **A global binder** across communities, plus per-event set codes so one person can
  appear in several sets.
- **The generative image step**, slotted into the single file where a card row is
  created: [`lib/services/cards.ts`](lib/services/cards.ts).
