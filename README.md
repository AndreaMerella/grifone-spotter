# Handoff: Grifone Spotter — Genoa CFC heritage hunting app

## Overview
**Grifone Spotter** is a community app for fans of **Genoa CFC** (Italy's oldest football club, founded 1893) to **find, photograph, and catalogue every depiction of a Grifone (griffin) in the city of Genova** — statues, civic crests on lampposts, marble plaques, ultras stickers, graffiti, building facades, etc. Think *Pokémon GO meets a heritage register, painted in rossoblù*.

Core loop:
1. Walk around Genova.
2. Spot a Grifone in the wild.
3. Open the app → photo → confirm what kind it is → pin drops on the map.
4. Earn badges, climb the *Classifica Ricercatori*, "take over" your quartiere.

## About the Design Files
The HTML files in `design/` are **design references** — high-fidelity React-in-HTML prototypes built to communicate the intended look, feel, and interactions. **They are NOT production code to ship.** Your job is to **recreate these designs in the target codebase's environment** (React Native / Expo recommended for a real mobile app, or a PWA with React + Vite), using its established patterns, libraries, and design system. If no environment exists yet, choose the most appropriate stack for a mobile-first geolocated photo app.

## Fidelity
**High-fidelity.** Final colors, typography, spacing, copy, iconography and component states are intentional. Recreate pixel-perfectly where possible, substituting real photos for placeholder photo blocks and a real map provider (Mapbox / MapLibre) for the Leaflet+OSM stand-in.

## Recommended Stack
- **Mobile**: React Native + Expo (iOS + Android, single codebase) — or PWA (React + Vite + Workbox) for fastest path
- **Backend**: Supabase (Postgres + PostGIS + Auth + Storage + Realtime) — matches the app's needs almost 1:1
- **Maps**: Mapbox GL or MapLibre (the design uses dark/paper/satellite styles)
- **Image storage**: Supabase Storage or Cloudinary (with image transforms for thumbs)
- **AI scan (optional v2)**: Cloud Vision API / Replicate / a fine-tuned classifier on collected Grifone photos
- **Auth**: Supabase Auth (email + Apple/Google)

## Data Model (suggested Postgres / Supabase schema)

```sql
-- Variants of the Grifone (5 historical types — see VARIANTS in src/data.js)
CREATE TABLE grifone_variant (
  id text PRIMARY KEY,           -- 'medievale' | 'liberty' | 'fascista' | 'moderno' | 'street'
  name text NOT NULL,
  era text NOT NULL,             -- 'XII–XV sec.' etc.
  description text,
  rarity text                    -- 'comune' | 'raro' | 'leggendario'
);

-- Subject types (statue, plaque, sticker, etc.)
CREATE TABLE subject_type (
  id text PRIMARY KEY,           -- 'statua' | 'stemma' | 'graffiti' | 'targa' | 'curiosita' | 'sticker'
  name text NOT NULL,
  icon text
);

-- Quartieri (neighborhoods of Genova)
CREATE TABLE quartiere (
  id serial PRIMARY KEY,
  name text NOT NULL,
  bounds geography(Polygon, 4326)
);

-- Users (ricercatori)
CREATE TABLE hunter (
  id uuid PRIMARY KEY REFERENCES auth.users,
  username text UNIQUE NOT NULL,
  display_name text,
  avatar_url text,
  joined_at timestamptz DEFAULT now()
);

-- The pins themselves
CREATE TABLE grifone_pin (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  variant_id text REFERENCES grifone_variant,
  subject_id text REFERENCES subject_type,
  location geography(Point, 4326) NOT NULL,    -- PostGIS for "near me" queries
  quartiere_id int REFERENCES quartiere,
  photo_url text NOT NULL,
  thumb_url text,
  caption text,
  found_by uuid REFERENCES hunter,
  found_at timestamptz DEFAULT now(),
  verified boolean DEFAULT false,              -- crowd / mod verification to fight trolls
  verify_count int DEFAULT 0,
  flagged boolean DEFAULT false
);

CREATE INDEX ON grifone_pin USING GIST (location);

-- Badges
CREATE TABLE badge (id text PRIMARY KEY, name text, description text, icon text, threshold int);
CREATE TABLE hunter_badge (
  hunter_id uuid REFERENCES hunter,
  badge_id text REFERENCES badge,
  awarded_at timestamptz DEFAULT now(),
  PRIMARY KEY (hunter_id, badge_id)
);

-- Verifications (crowd-sourced anti-troll)
CREATE TABLE pin_verification (
  pin_id uuid REFERENCES grifone_pin,
  hunter_id uuid REFERENCES hunter,
  vote text CHECK (vote IN ('confirm', 'reject', 'wrong_variant')),
  created_at timestamptz DEFAULT now(),
  PRIMARY KEY (pin_id, hunter_id)
);

-- Comments (optional)
CREATE TABLE pin_comment (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  pin_id uuid REFERENCES grifone_pin,
  hunter_id uuid REFERENCES hunter,
  body text NOT NULL,
  created_at timestamptz DEFAULT now()
);
```

### Seed data
The mock arrays in `design/src/data.js` (`VARIANTS`, `SUBJECTS`, `QUARTIERI`, `BADGES`) are real, curated content — copy them directly into your seed migration. The `PINS`, `HUNTERS`, `ACTIVITY` arrays are fake demo data and should NOT be seeded; they exist only to make the prototype feel alive.

## API Endpoints (REST or Supabase RPC)

| Method | Path | Purpose |
|---|---|---|
| `POST` | `/auth/signup` | Email + Apple/Google |
| `GET`  | `/pins?bbox=...` | Pins inside a map bounding box (clustered if >N) |
| `GET`  | `/pins/near?lat&lng&radius` | Pins near user (capture screen "you might be at…") |
| `GET`  | `/pins/:id` | Pin detail + comments + verification status |
| `POST` | `/pins` | Create pin (multipart: photo + json body) |
| `POST` | `/pins/:id/verify` | Vote confirm / reject / wrong_variant |
| `POST` | `/pins/:id/comments` | Add comment |
| `GET`  | `/me/collection` | My pins (Sala dei Trofei) |
| `GET`  | `/me/badges` | My badges + progress |
| `GET`  | `/leaderboard?period=week\|month\|all` | Top ricercatori |
| `GET`  | `/quartieri/:id/stats` | Pin count, top hunter, takeover % |
| `GET`  | `/feed` | Recent activity (for desktop heatmap right rail) |

## Screens / Views

All screens are in **Italian** by default with optional **Zenese** (Genoese dialect) and **English** variants — see `I18N` object in `src/data.js` for the full string table.

### 1. Onboarding (`OnboardingScreen`)
- **Purpose**: First-launch hype reel. 4 cards: welcome → what's a Grifone → how to hunt → grant permissions.
- **Layout**: Full-bleed phone screen, navy background `#011E41`, large Anton-display headers, paginated dots at bottom, "ENTRA IN CACCIA" red CTA.
- **Hero element**: Animated `<GrifoneLiberty>` SVG, scaled large.

### 2. Home / Map (`HomeMap`)
- **Purpose**: The main view. See all Grifoni pinned across Genova on a real map.
- **Layout**:
  - Full-screen Leaflet map with custom tile style (dark default, paper or satellite via Tweaks)
  - **Top bar**: "scarf" — diagonal red/blue stripes, Anton-display "GENOVA" + "TROVATI: N", search icon
  - **Filter chips row**: All / Statue / Stemmi / Graffiti / Targhe / Sticker — chip turns rossoblù when active
  - **Bottom-center FAB**: 72px red circle, white camera icon, drop-shadow — opens Capture flow
  - **Map pins**: Custom rossoblù markers with variant icon. Cluster at zoom < 14.
  - **Tab bar** (5 tabs, see Tab Bar below)
- **Real-data behaviour**: Fetch pins inside current bounding box on pan/zoom. Debounce 300ms.

### 3. Capture flow (`CaptureScreen`)
A 3-step flow inside one screen with progress dots.
- **Step 1 — Photo**: Camera viewfinder (real device camera; simulated as red-tinted block in mock). Big white shutter button. "Annulla" top-left.
- **Step 2 — Conferma**: Photo at top, then **subject type** picker (6 chips: Statua/Stemma/Graffiti/Targa/Curiosità/Sticker), then **variant** picker (5 cards: Medievale/Liberty/Fascista/Moderno/Street, each with its SVG silhouette + era label + rarity tag). Show GPS auto-detected quartiere.
- **Step 3 — Pin dropped**: Success animation, "FORZA GRIFO!", points awarded, badges unlocked. CTA "VEDI SULLA MAPPA" / "TROVA UN ALTRO".

**Anti-troll rules** (server-side):
- Photo must be < 60s old (EXIF check)
- GPS must be within Genova bounding box (or any city you expand to)
- Pin starts unverified; needs 3 confirms from other users to be verified
- Variant classifier (server-side, optional v2) validates user's variant pick — if confidence < threshold and disagreement, flag as "needs review"

### 4. Grifone detail (`DetailScreen`)
- Hero photo full-bleed
- Variant badge (red pill, Anton-display)
- Title: variant name + subject type (e.g. "GRIFONE MODERNO · Statua")
- Meta: quartiere, address, found by @username, date, verify count
- "Conferma" / "Segnala" actions
- Comments list
- Map preview at bottom

### 5. Sala dei Trofei (`CollectionScreen`)
- **Purpose**: My personal grifone catalogue
- **Layout**: Header "SALA DEI TROFEI · N/total catturati", then variant-by-variant accordion. Each variant shows progress bar (e.g. 3/12 found) and a grid of caught Grifoni (photo thumbs) + ghost slots for ones still missing.

### 6. Profile (`ProfileScreen`)
- Avatar, username, "GRIFONE D'ORO" honorific if earned
- 3-stat row: Catturati / Quartieri / Punti (large Anton-display numbers)
- Badge grid (8–12 badges with locked/unlocked states)
- "Modifica profilo" / "Impostazioni"

### 7. Classifica (`LeaderboardScreen`)
- Tabs: Ricercatori / Quartieri / Settimana / Mese / All-time
- Ricercatori list: rank · avatar · username · pin count · variant breakdown bar
- Quartieri list: rank · quartiere name · pin count · "takeover %" with rossoblù fill bar

### 8. Desktop Heatmap (`DesktopDashboard`)
- 3-column dashboard, dark theme, for partners / press / curva leadership
- **Left rail**: Stemmario stats — total Grifoni, breakdown by variant (5 rows with SVG + count + bar), this week's growth
- **Center**: Big rossoblù-tinted map of Genova with all pins, heatmap overlay toggle
- **Right rail**: Live activity feed (latest captures, scrolling) + Top 10 ricercatori

## Components (UI primitives in `src/`)

- `<IOSDevice>` — phone bezel for the desktop preview only; **drop in production**, the real app fills the device viewport.
- `<TabBar>` — bottom nav, 5 tabs: Mappa / Trofei / Cattura / Classifica / Profilo. The middle Cattura tab is the elevated red FAB.
- `<GrifoneMedievale>` / `<GrifoneLiberty>` / `<GrifoneFascista>` / `<GrifoneModerno>` / `<GrifoneStreet>` — 5 SVG silhouettes representing the variants. **You can keep these SVGs as-is** — they were custom-drawn for this concept and are not copied from any club asset.

## Design Tokens

### Color
- **Rosso Genoa** `#E5002A` — primary action, accents, fill of "takeover"
- **Blu Marina**  `#011E41` — primary background (mobile)
- **Carta**       `#F4EBD0` — light background option (paper mode)
- **Oro**         `#D4AF37` — secondary accent (subtitles, "GRIFONE D'ORO")
- **Off-white**   `#FAF7F1` — card surfaces on light bg
- **Inchiostro**  `#0A0F1C` — deepest text on light, deepest bg on desktop
- **Stripe pattern** — `repeating-linear-gradient(90deg, #E5002A 0 60px, #011E41 60px 120px)` used as the "scarf" header on map.
- **Intensity tweak** — `t.intensity` (0–100) scales the saturation of all rossoblù elements; in production this becomes a user accessibility setting.

### Typography
- **Display**: `Anton, Impact, sans-serif` — uppercase, tight tracking, used for all titles and chip labels. Conveys ultras / curva energy.
- **Body / UI**: `Inter, -apple-system, system-ui, sans-serif` — 14–16px, regular/medium.
- **Numerals (stats)**: Anton, 32–48px.
- **Mono accent**: `JetBrains Mono` for tiny meta text (coordinates, timestamps).

### Spacing
4px base scale: 4, 8, 12, 16, 20, 24, 32, 48, 64.

### Radius
- Buttons / chips: 4–6px (intentionally low — feels official, not soft)
- Cards: 8–12px
- FAB: full circle
- Pin markers: full circle

### Shadow
- FAB: `0 6px 20px rgba(229,0,42,0.5)`
- Cards over map: `0 4px 14px rgba(0,0,0,0.4)`

## Interactions & Behavior

- **Map pan/zoom** → debounced 300ms `/pins?bbox=` fetch, replace markers
- **Tab switch** → `setScreen(id)`; the Cattura center tab opens Capture full-screen modal
- **Pin tap** → bottom-sheet preview, tap again → full Detail screen
- **Capture flow** → cancellable at any step (preserves photo if user backs up from step 2)
- **Verify** → optimistic UI update, rollback on server error
- **Onboarding** → only on first launch; persist `hasOnboarded` in AsyncStorage / localStorage
- **i18n** — language stored in user prefs; Zenese is dialect (e.g. "Zena" for Genova); English for tourists/expats

## State Management
- Auth state → context / Zustand
- Map viewport (center, zoom) → URL params or local state
- Visible pins → React Query (`useQuery(['pins', bbox])` with 30s stale time)
- My collection / badges / leaderboard → React Query, refetch on focus
- Capture draft (in-progress photo + variant pick) → ephemeral local state; clear on submit/cancel

## Assets
- **Map tiles**: replace OSM with Mapbox style (custom rossoblù-tinted dark map recommended). Free MapLibre alternative.
- **Photos**: All photo blocks in the mock are placeholders. Replace with real Supabase Storage URLs.
- **Icons**: Use Phosphor / Lucide / Tabler. The mock uses inline SVG glyphs.
- **Grifone SVGs**: ship the 5 from `src/grifoni.jsx`.
- **Logos**: Do NOT use the official Genoa CFC crest in production without a license. Use the variant SVGs (which are original interpretations) as the in-app logo.

## Privacy / Legal
- Genoa CFC trademarks: this is a fan project. Either rename ("Cerca-Grifone", "Genova Heritage", etc.) and stay decorative, OR pursue an official partnership with the club. The README's wording assumes fan-project-with-future-partnership-aspiration.
- GPS / camera: standard iOS/Android permission flow.
- Photo moderation: ship NSFW + face-blur on uploads (the curva is a public space; faces of bystanders should be obscured).
- Anti-troll: see verification rules above. Add report button on every pin.

## Files
- `design/Grifone Spotter.html` — entry point. Open in any browser.
- `design/src/app.jsx` — single-window root, view router, Tweaks defaults.
- `design/src/data.js` — VARIANTS, SUBJECTS, QUARTIERI, BADGES (real content), PINS/HUNTERS/ACTIVITY (mock content).
- `design/src/grifoni.jsx` — 5 SVG variants.
- `design/src/mobile-map.jsx` — Leaflet integration + filter chips + FAB.
- `design/src/mobile-screens.jsx` — Onboarding, Capture, Detail, helpers.
- `design/src/mobile-screens-2.jsx` — Collection, Profile, Leaderboard.
- `design/src/desktop.jsx` — `<DesktopDashboard>` 3-col heatmap.
- `design/src/ios-frame.jsx` — phone bezel; **drop in production**.
- `design/src/tweaks-panel.jsx` — design-time controls; **drop in production**.
- `design/PRD.md` — one-page PRD (paste into any LLM).

## Recommended Build Order (MVP, ~3–4 weeks for one dev)
1. **Week 1** — Auth, schema, seed VARIANTS/SUBJECTS/BADGES, basic React Native shell with Map screen.
2. **Week 2** — Capture flow (camera → upload → POST /pins), Pin Detail.
3. **Week 3** — Sala dei Trofei, Profile, Badges, Leaderboard.
4. **Week 4** — Onboarding, polish, language toggle, anti-troll verification UI.
5. **v2** — AI variant classifier, comments, social follow, quartiere takeover events.

## Open Questions for Product
- Does this stay strictly Genoa CFC fans, or open up to "all heraldic griffins in Italy / Europe"?
- Monetization: free with optional club-partnered merch drops? Premium "Ricercatore d'Oro" tier?
- Is moderation crowd-sourced or do you have a small ultras-led mod team?
- Real-world meetups / events ("ricerca di gruppo" days) — surface in the app or out of scope for v1?
