# The First Release — scroll-driven campaign page (concept prototype)

Single-product fashion experience for an off-white hoodie carrying the Urdu
calligraphy of Jaun Elia's name. Built by OSAV. Not yet deployed.

## Backend

Next.js route handlers + SQLite through Node's built-in `node:sqlite`
(Node 22.13+; no extra packages). Settings: copy `.env.example` to `.env.local`.

| Route | What it does |
| --- | --- |
| `GET /api/stock` | Live edition count `{ total, remaining, live }`; every counter on the site reads it |
| `GET /api/checkout` | Whether ordering is open, payment methods, flat shipping fee |
| `POST /api/orders` | Places an order (cash on delivery). Re-prices from `content/products.ts`, reserves pieces atomically, idempotent per checkout |
| `GET /api/orders/:code?t=` | The customer's own order (private token from the confirmation link/email) |
| `POST /api/contact` | Contact form → stored for /admin, emailed to `TEAM_INBOX_EMAIL` |
| `POST/DELETE /api/subscribe` | Release list ("Notify me") |
| `/api/admin/*` | Orders, status changes (cancel restocks), shipping per order, messages, release list CSV, stock correction |

- **/admin** — order desk (password `ADMIN_PASSWORD`; off when unset).
- **/order/[code]** — the customer's order page and status.
- **Email** — via Resend when `RESEND_API_KEY` + `MAIL_FROM` are set; otherwise printed to the server log.
- **Data** — `data/store.db` (git-ignored). Needs a persistent disk: fine on a VPS/Render/Railway/Fly; on
  Vercel swap `src/server/db.ts` for a hosted database (all SQL lives in `src/server`).
- **Not built** — online card/wallet payments (needs a merchant account and keys; COD only for now).
- **Launch** — set `NEXT_PUBLIC_SITE_URL` and `NEXT_PUBLIC_SITE_INDEXABLE=true`, or the site stays out of search.

## Run

```bash
npm install
npm run dev        # http://localhost:3000
npm run lint
npx tsc --noEmit
npm run build && npm start
```

The full experience runs for every visitor, including machines with OS
animations turned off (that setting previously disabled the hero video and the
rotation — see `src/lib/motion.ts`). Add `?motion=off` for a static layout.

## The six scenes

| Scene | What happens | Driven by |
| --- | --- | --- |
| 01 The reveal | Emergence footage full-bleed, autoplays once (muted, inline, no controls), holds its last frame. "WORDS / Made / WEARABLE" rise out of masks around the garment (difference blend). On scroll the footage closes to a portrait window, the words part, and the ivory of scene 02 rises like a curtain. | `HeroVideo`, scroll scrub |
| 02 The words | "WORDS / That / STAY" slide in from opposite sides, cross and lock; the chest close-up opens as a window; lines drift apart; the page darkens into the studio. | scroll scrub |
| 03 In motion | The real rotation **video** is scrubbed by scroll: `currentTime = progress × duration`. Forward turns it, back reverses it. Degree readout, Front/Profile/Back list, progress rail and a ghost word behind the garment follow the same scroll position. | `VideoScrub`, scroll |
| 04 The details | Images open from a sliver and drift at different speeds; annotations rise in; the detail footage is scrubbed by its own pass through the viewport. | scroll |
| 05 The back | The back is drawn down like ink; the title arrives; leader lines name each part from outside the garment. | scroll scrub |
| 06 The final frame | Bookend: the garment settles into the light while "The story / starts / here." assembles around it. | scroll scrub |

All choreography is in `src/components/scenes/Choreography.tsx` (GSAP + ScrollTrigger,
native scrolling, no hijacking). Default CSS is the resolved composition, so the page
reads correctly with no JS.

## Rotation: real video scrubbing (`VideoScrub`)

`public/media/rotation-scrub.mp4` is the supplied rotation re-encoded for seeking:
720×1280, 8.00 s, 24 fps, H.264 with a keyframe every 4 frames and no B-frames
(1.7 MB). Any seek decodes at most 4 frames, forwards or backwards.

- Waits for `loadedmetadata`, then sets `currentTime = progress × duration`, where
  progress is the scroll position inside the product section only.
- One seek in flight at a time; the next target is applied on `seeked`, so seeks
  never pile up and fast scrolls land on the latest position.
- Never plays on its own; any stray `play` is paused. iOS is primed with a muted
  play/pause so it decodes frames for seeking.
- `preload="metadata"` until the section is about a screen away, then `auto`.
- Measures seek times: if a browser's median seek exceeds 140 ms, a seek stalls,
  or the video errors, it swaps to `FrameSequence` (WebP frames extracted from the
  same footage, `public/frames/rotation`).

The detail footage still uses `FrameSequence` (`public/frames/details`).

## Editing content

`src/content/site.ts` holds all copy, asset paths and switches:
`flags.showProposedMark` (OSAV's proposed ن mark — not approved),
`flags.showCommercials` (2,500 · PKR 20,000 — hidden per the brief),
`flags.showReleaseComingSoon`. All display copy is provisional.

## Known limitations of the supplied assets

1. **Emergence video artwork is wrong** — calligraphy large and centred (the product has
   it small on the left chest), "JAUN ELIA" garbled. It opens the page; regenerate it.
2. **Rotation video** is an AI render: the chest mark is larger than on the reference,
   and the back lettering softens mid-turn. Real product rotation (turntable shoot or a
   GLB) would fix both.
3. **Detail footage** is PixVerse output: watermark in the source (cropped), tan rather
   than ivory, and its final seconds show a hoodie with no artwork (trimmed). Check
   PixVerse's commercial terms or replace with real macro photography.
4. **Back verse** contains malformed lines (also flagged in OSAV's brand guidelines).
   Shown as supplied, marked unverified.
5. The degree readout assumes the rotation video turns at a constant speed.
