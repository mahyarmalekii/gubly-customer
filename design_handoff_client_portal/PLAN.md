# Gubly Client Portal — Build Plan

A client-facing portal, separate from the main gubly site but using the exact same "Sugar Pop" design system. One codebase, three surfaces, connected by a per-client link.

---

## 0. Design system (reuse, don't reinvent)

Copy from `gubly-site`:
- `candy-styles.css` tokens: `--c1:#F2BE45` butter · `--c2:#A9CFE8` sky · `--c3:#EE8B4F` tangerine · `--c4:#C4D89B` pistachio · `--pine:#2C5E3F` · `--paper:#F5F0E2` · `--card:#FFFDF4` · `--ink:#241F15` · radius `--r:30px`
- Fonts: **Fredoka** (display) + **Nunito** (body), Baloo 2 / Gluten as accents
- Components: pill `.btn` (btn-dark / btn-paper), rotated `.sticker` pills, floating pill `.topbar`, color-block `.hcard` cards, soft shadows, cream doodle SVGs
- Assets: `assets/mascots/*.png` (chef, cup, megaphone, bell, etc.), `gubli-logo.png`, wave graphics
- EN/DE language toggle (same `.lang` pill component as main site)

---

## 1. Architecture & link flow

```
you (admin)                     client
────────────                    ────────────
Admin Dashboard  ──creates──►   /c/{slug}?t={token}   (magic link, no login)
                                    │
                                    ├─ phase: "moodboard"  → Moodboard experience
                                    └─ phase: "live"       → Project Hub
```

- **Per-client record** keyed by slug (e.g. `/c/cafe-luna`). Token in URL = auth (magic link, no password).
- Client record has a `phase` field; the same link shows moodboard first, then flips to the hub when you advance the project. Client never sees a dead link.
- Hosting: Firebase (already used — `.firebaserc` exists). Data: **Firestore** (clients, responses, analytics events). No backend server needed.

### Data model (Firestore)
```
clients/{slug}
  name, type (café|restaurant|bar|bakery|retail), lang, phase,
  token, createdAt, lastVisit, visitCount
clients/{slug}/moodboard/…      // the options you curated for them
clients/{slug}/responses/…      // reactions, rankings, comments, picks
clients/{slug}/analytics/…      // dwell + click events (batched)
clients/{slug}/hub/…            // posts, events, promos, timeline, files
```

---

## 2. Surface A — Client Moodboard (`/c/{slug}`)

Warm personal landing: "Hi Luna Café! 👋 Mahyar picked these for you" — mascot, sticker, big Fredoka headline. Then **5 sections** (one per category, scroll journey with progress pill):

1. **Style directions** — 3–4 full website-look cards (screenshot/mock in a browser-frame card). Reactions: ❤️ Love / 👍 Like / 🙅 No thanks (sticker-style buttons), then **drag-to-rank** favorites strip.
2. **Color palettes** — swatch cards (each a candy color-block card). Pick-one + optional comment.
3. **Typography** — font-pair specimen cards ("Aa" big, sample sentence). Pick-one.
4. **Reference websites** — link cards with preview image + "what do you like about it?" comment field per item.
5. **Imagery mood** — photo grid, tap to love (heart sticker pops on).

Per-item **comment bubble** (small tangerine speech-bubble button on every card).
Final step: summary of their picks → confirm & send → celebratory mascot screen.

**Interaction rules:** every reaction saved immediately (no submit-or-lose); progress persisted so they can leave and return.

### Analytics captured invisibly
- **Dwell time per item** — IntersectionObserver ≥50% visible + tab-visible; accumulate seconds per item id.
- **Click/hover heat** — throttled mousemove + click coords, normalized to page width; batched to Firestore every 10s / on unload (sendBeacon).
- **View sequence** — order items entered viewport; revisit counts before final pick.
- **Session** — visitCount++, lastVisit, device, total duration.

---

## 3. Surface B — Admin Dashboard (`/admin`)

Simple password gate (it's just you). Sections:

1. **Client list** — cards: name, type mascot, phase chip, last visit, "copy link" button.
2. **New client wizard** — name, type, lang → pick moodboard content per category (from your reusable library + custom uploads) → generates link → copy/QR/WhatsApp share.
3. **Results view (per client)** —
   - Picks summary: winner per category, rankings, all comments threaded per item.
   - **Heatmap tab**: page screenshot of their moodboard with a canvas heat overlay (blur+gradient from click/hover points); toggle clicks vs. hovers.
   - **Attention tab**: bar chart of dwell seconds per item, "viewed 6× before choosing" badges, visit count + last visited.
4. **Advance phase** button: moodboard → in-progress → live hub. Same link, new experience.
5. **Hub content manager**: add posts (draft → client approves), events, promotions, timeline steps, upload files.

---

## 4. Surface C — Client Project Hub (same link, after flip)

Dashboard-style, candy cards on cream:

- **Hero**: "Your project, {name}" + status sticker + project **timeline** (Kickoff → Design → Build → Launch, pine progress).
- **Website preview**: browser-frame card, live iframe/screenshot of their site + "visit" button.
- **Posts to approve**: social-post cards (image + caption, EN/DE) with Approve ✓ / Request change ✎ — approvals notify you.
- **Upcoming events**: date-block cards (big Fredoka date numeral).
- **Promotions**: coupon-style ticket cards (dashed edge, sticker badge).
- **Files & deliverables**: logo pack, brand guide, photos — download list.
- Everything mascot-flavored per client type (café → cup mascot, bakery → cake, bar → cheers…).

---

## 5. Build order (for the coding model)

1. Scaffold: shared `portal.css` (fork candy-styles), copy assets, Firebase init, i18n dictionary EN/DE.
2. Client record loader + phase router at `/c/{slug}`.
3. Moodboard UI (all 5 categories) with local persistence first.
4. Analytics capture layer (dwell, heat, sequence) — pure JS module, batched writes.
5. Admin: client list + wizard + copy-link.
6. Admin results: picks summary → dwell bars → heatmap canvas overlay.
7. Project Hub + admin hub content manager.
8. Polish: mascot moments, confetti on submit, empty states, mobile (clients will open the link on phones — mobile-first for surfaces A & C).

## 6. Variations worth exploring (pick during design)

- Moodboard as **scroll journey** (all sections, one page) vs. **step-by-step wizard** (one category per screen, Typeform-like) vs. **card-deck swipe** (Tinder-style on mobile).
- Heatmap as **overlay on screenshot** vs. **per-item glow intensity** on a miniature of the page (simpler, more readable).
- Hub as **single dashboard** vs. **tabbed sections**.
