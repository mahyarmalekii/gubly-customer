# Handoff: Gubli Client Portal (Moodboard · Project Hub · Admin Dashboard)

## Overview
A client-portal product for **gubli** (Berlin gastronomy & retail digital services). The designer creates a per-client magic link. The client opens it and sees a **Vibe Check** board (picks style direction, colors, fonts, reacts to reference sites, hearts photo moods, adds their own links/photos/videos). The designer watches results in an **Admin Dashboard** (picks summary, per-section attention/dwell time, click heatmap, visit stats), then flips the client's phase so the **same link** becomes a **Project Hub** (website draft preview, social posts to approve, events, promotions, project timeline, file deliverables).

## About the Design Files
The `.dc.html` files in this bundle are **design references created in HTML** — working prototypes showing intended look and behavior, not production code to ship. Recreate these designs in your chosen stack. Recommended stack (chosen by the client): **Firebase** — Firestore + Firebase Hosting + Cloud Functions where needed. Frontend framework is your choice (the gubly main site is static HTML/CSS; React or plain JS both work — the designs are simple enough for either).

## Fidelity
**High-fidelity.** Colors, typography, spacing, radii, copy (EN + DE) and interactions are final. Recreate pixel-perfectly.

## Build priority

**Phase 1 (build now): Vibe Check + push reports.** The owner does NOT want to maintain an admin UI. Results are pushed to him instead:

- **On client submit** → Cloud Function `onSubmit`:
  - **Telegram**: Bot API `sendMessage` to the owner's chat id — a formatted summary (picks, rankings, all comments, custom items with links) + `sendPhoto` with a server-rendered heatmap image (render the miniature page + radial heat dots on a node-canvas, or send the raw click coords rendered client-side at submit time and uploaded to Storage).
  - **Email**: Firebase Extension "Trigger Email" (or Resend) — same content as HTML email, heatmap attached, plus attention stats table (dwell seconds per section) and client uploads embedded/linked.
  - **Desktop note on laptop**: subscribe the owner's laptop to an **ntfy.sh topic** (free, no account); the same function POSTs to `https://ntfy.sh/<private-topic>` → native desktop notification.
- **On each visit** → `getPortal` function fires a short, silent Telegram message ("👀 Luna Café opened their Vibe Check — visit #4, on mobile") and an ntfy ping. No email for visits.
- Store everything in Firestore anyway (responses + analytics as specced below) so future phases can read it.

**Client creation is automated — no admin UI.** The owner creates a client by sending ONE message to his own Telegram bot ("new Luna Café, Selin, cafe, de") or filling the one-field Link Maker (see `Link Maker.dc.html` for the design). A Cloud Function parses it, creates `clients/{slug}` with a random token, and replies with the ready-to-send link (`https://<domain>/c/{slug}?t={token}`). Everything on the Vibe Check page personalizes automatically from the client record: business name (topbar, headline stickers, font specimens), contact first name (greeting), business type → hero mascot (cafe→cup, bakery→cake, bar→cheers, restaurant→chef, retail→vip) and default language. The prototype demonstrates this with URL params (`?name=&contact=&type=&lang=&c=`) — in production these come from Firestore via the token lookup, and per-client state is keyed by slug.

**Phase 2 (later, optional): Project Hub + Admin Dashboard.** The designs exist (`Project Hub.dc.html`, `Admin Dashboard.dc.html`) and the data model already supports them — do not build these until asked. Nothing in phase 1 may depend on them.

## Backend & Auth (Firebase) — build spec

### Data model (Firestore)
```
clients/{slug}                     // slug = short handle, e.g. "luna"
  name        "Luna Café"
  contactName "Selin"
  type        "cafe" | "restaurant" | "bar" | "bakery" | "retail"
  lang        "en" | "de" | "both"
  phase       "moodboard" | "design" | "live"   // controls what the link shows
  token       random 128-bit string             // magic-link secret
  createdAt, lastVisit, visitCount

clients/{slug}/moodboard/{itemId}  // curated content incl. custom items
  kind   "style" | "palette" | "font" | "ref" | "photo" | "link" | "video"
  title, url, meta…, addedBy ("admin" | "client")

clients/{slug}/responses/main
  reactions   { styleA: "love"|"like"|"no", … }
  loveOrder   ["styleA", …]                    // rank = first-love order
  palette, font                                // picked ids
  refLikes    { ref1: true, … }
  comments    { itemId: "text", … }
  photoLoves  { p1: true, … }
  submittedAt

clients/{slug}/analytics/{sessionId}
  dwell   { styles: 84, colors: 41, … }        // seconds per data-track section
  clicks  [[x,y], …]                           // normalized 0–1 page coords
  device, startedAt, duration
```

### Auth: magic link
- Client URL: `https://<domain>/c/{slug}?t={token}`. No login for clients.
- Implement with **Firebase custom claims OR plain token check**: simplest robust version is a **Cloud Function** `getPortal(slug, token)` that verifies token server-side and returns the client doc + moodboard items; all client writes (responses, analytics, custom items) go through callable functions that re-verify the token. This keeps Firestore rules simple: deny all direct client access.
- Admin: **Firebase Auth** (email+password or Google sign-in) restricted to the owner's UID. Firestore rules: `allow read, write: if request.auth.uid == OWNER_UID`.
- Advancing phase = admin updates `clients/{slug}.phase`; the client link re-routes automatically on next load.

### Analytics capture (client side, invisible)
- **Dwell**: IntersectionObserver on `[data-track]` sections (threshold 0.4) + `document.hidden` check; accumulate seconds in a map; flush every 10 s and on `visibilitychange`/`pagehide` via `navigator.sendBeacon` to a Function endpoint.
- **Click heat**: capture-phase click listener; store `[pageX/scrollWidth, pageY/scrollHeight]` normalized pairs; batch with dwell flush. (Prototype also shows hover capture as optional — clicks alone proved sufficient.)
- **Visits**: increment `visitCount`, set `lastVisit` in the `getPortal` function.

### Media
- Client photo uploads (moodboard photo slots, custom photo items) → **Firebase Storage** under `clients/{slug}/uploads/`; store the download URL on the moodboard item. Downscale client-side to ≤2048px WebP before upload (prototype's `image-slot.js` shows the canvas-downscale approach).
- Videos are **not uploaded** — clients paste YouTube/Vimeo URLs; embed via `youtube.com/embed/{id}` / `player.vimeo.com/video/{id}` (parse both `watch?v=`, `youtu.be/`, `shorts/`).

## Screens / Views

### 1 · Vibe Check (client board) (`Vibe Check.dc.html`) — route `/c/{slug}` when phase = "moodboard"
- **Topbar**: fixed, centered pill (max-width 1080px), bg `#FFFDF4`, 1.5px border `rgba(36,31,21,.10)`, radius 999px, shadow `0 6px 22px rgba(36,31,21,.07)`. Left: gubli logo 34px + "gubli × {clientName}". Center: anchor nav (Styles/Colors/Fonts/Sites/Photos/Extras). Right: EN/DE segmented toggle (active segment `#241F15` bg, `#FFFDF4` text).
- **Hero**: card radius 40px, bg `#F2BE45` (butter), padding 44–48px. Sticker pill rotated −2°, bg `#2C5E3F`, text `#FBF2D8`, pulsing 8px dot `#9FE08A`. H1 Fredoka 600 uppercase `clamp(52px,8vw,110px)`, line-height .96; "vibe" in `#2C5E3F`, "!" in `#EE8B4F`. Floating cup mascot bottom-right, 5s ease float animation. CTA pill button bg `#2C5E3F` text `#FFFDF4`, hover lift −2px + shadow.
- **Marquee ribbon**: full-width, bg `#2C5E3F`, rotated −1.2°, Fredoka 600 15px uppercase, cream text, `✷` separators in `#F2BE45`. (Static in prototype; can scroll slowly.)
- **Section pattern** (all 6 sections, max-width 1080px): numbered sticker pill (rotating ±2°, section accent color), H2 Fredoka 600 uppercase `clamp(36px,4.6vw,64px)` with one word in `#2C5E3F`, Fredoka 500 15px sub-line at 64% ink.
- **01 Styles**: 6 cards in `grid auto-fit minmax(290px,1fr)`, gap 18px. Card: radius 30px, tinted bg (`#FBEABF`, `#E3EEF0`, `#FBDFC9`, `#EBF0D6`), 1.5px border, padding 20px; hover translateY(−4px) + shadow `0 14px 34px rgba(36,31,21,.09)`. Each holds a 4:3 mini website mock (radius 20px), name (Fredoka 600 22px uppercase), one-line description, and reaction pills: 😍 Love / 🙂 Like / 🙅 Not for me (selected = bg `#2C5E3F` text `#FBF2D8`) + 💬 36px round comment toggle revealing a textarea. First-loved card gets a 44px round `#1`-badge, rotated 8°, pop-in animation; rank = love order.
- **02 Colors**: 9 palette cards (auto-fit minmax(230px,1fr)): 4 overlapping 44px swatch circles, name + mood line; pick-one — selected card gets 2.5px `#2C5E3F` border, green shadow, "✓ your pick" sticker.
- **03 Fonts**: 7 specimen cards: label, client name rendered in the candidate font (Baloo 2 / DM Serif Display / Space Grotesk / Gluten / Archivo Black / Cormorant Garamond / Space Mono) at `clamp(32–34px, 3.4–3.6vw, 42–46px)`, sample sentence; pick-one, same selected treatment.
- **04 References**: full-width rows: 52px numbered circle (rotated −6°), site name + url + why, per-item textarea "What do you like about it?", right column "Visit ↗" pill + "❤ I like this one" toggle.
- **05 Photos**: grid auto-fill minmax(220px,1fr); square image slots (radius 24px) with 44px heart button top-right (loved = bg `#EE8B4F`, white heart) and label pill bottom-left. NOTE: `image-slot` custom element needs `width:100%;height:100%` forced via CSS.
- **06 Extras**: custom items grid + "Add your own" composer (dashed 2px border card): segmented kind picker (🔗 Link / 📷 Photo / ▶ Video), title input, URL input (hidden for photo), "Add to board ✓" button. Items render as link card (44px 🔗 circle + host + Visit pill), video card (16:9 iframe embed), or photo card (4:3 image slot); each has ♥ toggle, "added by …" label, and ✕ remove (top-right, −10px overhang).
- **Sticky bottom bar**: dark pill `#241F15`, "n / 5" progress + 5 dots (done = `#F2BE45`), "Send to Mahyar ✓" butter button.
- **Submit overlay**: dark scrim 45%, cream modal radius 40px, cheers mascot overhanging top (−100px), "Danke!" headline, links to Project Hub / keep editing.

### 2 · Project Hub (`Project Hub.dc.html`) — same route when phase = "live"
- Same topbar pattern. Hero card bg `#C4D89B` (pistachio) with laptop mascot; H1 "Your project, {clientName}".
- **Timeline pill**: Kickoff → Moodboard → Design → Build → Launch; done = green circle ✓, current = butter circle ★ with 2px green ring, future = grey number, connectors 34px lines.
- **Website preview**: browser-frame card (3 traffic dots in tangerine/butter/pistachio, url pill "lunacafe.berlin — draft v2") + companion butter card with "Open preview ↗" and "💬 Give feedback".
- **Posts to approve**: 3 cards, each: square visual (tinted bg + mascot + Baloo 2 caption), channel pill + date, caption text, "Approve" (green, → "✓ Approved" pistachio) + "✎" request-change; status sticker overhangs top (draft = butter / approved = green / change = tangerine). Header shows "{n} waiting" counter. Actions fire a dark toast.
- **Events**: date-block rows (64px white square: day 22px + month), name, meta, status sticker.
- **Promos**: coupon cards with 1.5px **dashed** border, Baloo 2 title, monospace-style code chip, usage stat, "live" sticker overhang.
- **Files**: full-bleed pine `#2C5E3F` card radius 44px, cream text, grid of white download cards (emoji icon 26px, name, meta, "Download ↓" in green).

### 3 · Admin Dashboard (`Admin Dashboard.dc.html`) — route `/admin`, Firebase Auth required
- **Dark topbar** variant: bg `#241F15`, cream text, "admin — only you" green sticker.
- **Client cards**: mascot 56px, name, type · lang, phase sticker (moodboard = butter / design = sky / live hub = green), visits + last visit, "🔗 Copy link" (writes to clipboard, toast) + `gub.li/{slug}` chip. Selected card = 2.5px green border. "+ New client link" green pill button (opens creation wizard: name, type, lang → curate moodboard → generate link).
- **Results panel**: white card radius 44px with 3 segmented tabs:
  - **Picks**: 4 winner cards (category label, winner, detail) + comment thread (34px butter initial circle, speech bubble bg `#F5F0E2` radius `4px 18px 18px 18px`).
  - **Attention**: horizontal dwell bars per section (26px rounded track `#F5F0E2`, fill in section accent color, seconds + behavioral note per row) + session stat chips (visits, total time, last visit, device).
  - **Heatmap**: 9:16 miniature of the moodboard page with radial-gradient heat dots (`rgba(238,139,79,.75)` core → transparent, mix-blend multiply, 28px) at normalized click coords; click-count chip; right column = insight cards + "↻ Refresh live data".
- **Advance phase** row: dashed top border, explainer text + tangerine "Advance to project hub →" button.

## Interactions & Behavior
- All moodboard reactions save **immediately** (no submit-or-lose); progress persists across visits. In the prototype this is `localStorage` (keys `gubli_resp_luna`, `gubli_dwell_luna`, `gubli_heat_luna`, `gubli_visits_luna`, `gubli_custom_luna`) — replace with Firestore writes as specced above. The Admin Dashboard reads these same keys, demonstrating the live data flow.
- Hover states: cards lift −3/−4px with soft shadow; buttons lift −2px. Transitions 0.15–0.2s ease.
- Sticker badges "pop in": `scale(.6)→1.12→1`, 0.3s.
- EN/DE toggle swaps all copy instantly (full dictionaries are in the prototypes' logic).
- Toasts: dark pill, bottom-center, auto-dismiss ~2.5s.
- Mobile: clients open links on phones — single-column stacking, hit targets ≥44px.

## Design Tokens (from gubly "Sugar Pop" system)
- Colors: butter `#F2BE45` · sky `#A9CFE8` · tangerine `#EE8B4F` · pistachio `#C4D89B` · pine `#2C5E3F` · paper `#F5F0E2` · card `#FFFDF4` · ink `#241F15` · cream text `#FBF2D8` · tints `#FBEABF` `#E3EEF0` `#FBDFC9` `#EBF0D6`
- Border: `1.5px solid rgba(36,31,21,.10)` (cards), `.16` (buttons/inputs)
- Radii: cards 30px, hero/panels 40–44px, inner media 18–24px, pills 999px
- Shadows: card hover `0 14px 34px rgba(36,31,21,.09)`, topbar `0 6px 22px rgba(36,31,21,.07)`, dark bar `0 14px 34px rgba(36,31,21,.28)`
- Type: **Fredoka** 500/600 (display, uppercase headings, UI labels) · **Nunito** 400–800 (body) · accents: Baloo 2, Gluten · specimen-only: DM Serif Display, Space Grotesk. Google Fonts.
- Sticker pattern: pill, Fredoka 600, 12–13px, uppercase, letter-spacing .08em, rotated ±2–8°, often overhanging card edges by ~10–12px.

## Assets
- `assets/gubli.png` — gubli blob logo (topbars).
- `assets/mascots/*.png` — hand-drawn ink mascots (transparent, recolored `#241F15`), cut from `assets/mascot-sheet.png`. Used: cup, cheers, laptop, moka, cake, flower, chef, megaphone (+ 7 spares).
- `image-slot.js` — drag-and-drop image placeholder web component used by photo slots (reference for the Storage upload UX).

## Files
- `Vibe Check.dc.html` — client board experience (personalizes via URL params: `?name=&contact=&type=&lang=&c=`)
- `Link Maker.dc.html` — owner-side link generator (design for the creation flow)
- `Project Hub.dc.html` — client project hub
- `Admin Dashboard.dc.html` — admin dashboard (results, heatmap, phases)
- `PLAN.md` — original architecture plan (link flow, phases, build order)
- `image-slot.js`, `assets/` — supporting assets

Open each `.dc.html` in a browser to see the live behavior. The `<x-dc>` template + `class Component` logic inside each file document the full interaction logic and all EN/DE copy.
