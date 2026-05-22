# Fixture Card Generator — Beta 0.114

A single-file HTML canvas app that generates printable AFL junior fixture cards from PlayHQ fixture pages. Built for Norwood Junior Football Club but designed to work for any EFNL (or other competition) club.

---

## Project history

This tool evolved through several iterations:
1. **Python/Playwright static HTML** — early versions generated fixed HTML cards per team, rendered to PNG via headless Chromium
2. **Canvas-based generator** — rebuilt as a single `index.html` that parses any PlayHQ fixture file and renders dynamically via Canvas 2D
3. **Current state** — fully dynamic, competition-aware, supports any club with asset folders, scales to A4 proportions

The font was changed from **Barlow Condensed** (early versions) to **Outfit** (current) for better rendering on canvas.

---

## How to use

1. In PlayHQ, open your team's fixture page
2. **File → Save Page As → Webpage, Complete** — save the `.html` file
3. Open `index.html` in a browser
4. Upload the saved `.html` file in the **Fixtures** tab
5. Adjust team details in the **Team** tab if needed
6. Select/deselect rounds in the fixture list
7. Click **Generate Card** → preview appears on the right
8. Export via **PNG** or **Print**

---

## Asset folder structure

All assets sit alongside `index.html`:

```
assets/
  badge.png              ← default competition badge (header fallback)
  wm_logo.png            ← default competition top-right watermark
  wm_monogram_bl.png     ← bottom-left club monogram watermark
  wm_player_kicking.png  ← greyscale player watermark (bottom-right)
  wm_player_running.png  ← greyscale player watermark (bottom-left)
  wm_huddle.png          ← greyscale team huddle watermark (centre)
  wm_tackle.png          ← greyscale tackle watermark (top-left)
  colours.json           ← club colour lookup table (keyed by club name)

  competitions/
    efnl/
      badge.png          ← overrides default badge for EFNL fixtures
      wm_logo.png        ← overrides default top-right watermark for EFNL
    mfl/
      badge.png
      wm_logo.png

  clubs/
    norwood/
      badge.png          ← club badge shown in card header
      crest.png          ← club crest shown in "our team" pill
      monogram.png       ← optional: club monogram for bottom-left watermark
    lysterfield/
      badge.png
      crest.png
    south_croydon/
      badge.png
      crest.png
    ...
```

### Competition folder slugs
League names → slugs via initials of significant words:
- `Eastern Football Netball League` → `efnl`
- `Mornington Football League` → `mfl`
- Short abbreviations like `EFNL` are used directly (lowercased)

### Club folder slugs
Club name lowercased, spaces → underscores:
- `Norwood` → `norwood`
- `South Croydon` → `south_croydon`
- `Upper Ferntree Gully` → `upper_ferntree_gully`
- `Glen Waverley Rovers` → `glen_waverley_rovers`
- `Beaconsfield Football Club` → `beaconsfield` (suffixes stripped)

---

## config.json

```json
{
  "club_name": "",
  "home_venue": "",
  "default_scheme": "purple"
}
```

Leave `club_name` empty for generic startup (no club-specific assets pre-loaded). If set, assets for that club load on startup before any fixture file is uploaded.

---

## Watermark image specs

All watermark PNGs should be:
- **RGBA** with transparent background
- **Greyscale** tones — renderer uses `source-over` at 20–30% opacity scaling with card lightness
- Player images: subjects isolated from background, no colour tinting

### Processing watermarks
The `wm_efnl.png` file was created from a palette PNG. **Important**: the project folder strips palette transparency from PNGs on save. To preserve transparency:
1. Rename the PNG to `.TXT` before uploading to the project
2. Process with Python: `img.convert('RGBA')` correctly handles palette transparency
3. Greyscale: `grey = (0.299*r + 0.587*g + 0.114*b)`, preserve alpha channel

For the EFNL badge watermark specifically, the green background oval was removed via HSV thresholding and exterior black was removed via flood-fill from edges.

---

## Card layout

- **Width**: `Math.max(560, Math.round(CARD_H * 0.707))` — A4 portrait proportions, min 560px
- **Height**: `PAD_TOP(24) + YEAR_BAR_H(28) + 4 + HEADER_H(96) + BETWEEN(12) + contentH + FINALS_H(68) + PAD_BOT(26)`
  - Each fixture row: `ROW_H(50) + ROW_GAP(4) = 54px`
  - Each bye row: `BYE_H(28) + ROW_GAP(4) = 32px`
- **Scale**: rendered at 2× (retina), displayed at 1×
- **Grid columns**: `COL_RND(24) + COL_DATE(52) + COL_VS(scales) + COL_GAP(scales) + 2×COL_PILL`
  - `COL_VS` and `COL_GAP` scale with `CARD_W / 560` so VS circle has breathing room on wider cards

### Year bar
- Sits at the very top of the card, centred
- Shows `"[YEAR] FIXTURES"` e.g. `"2026 FIXTURES"` in the secondary colour
- Year is extracted from the team name field at render time (`/\b(20\d{2})\b/`)
- Falls back to just `"FIXTURES"` if no year is found
- Team name on the card has the year stripped out

### Watermark progressive hiding
- `CARD_H ≥ 680px`: all 4 watermarks shown
- `560–679px`: running + kicking dropped
- `440–559px`: tackle also dropped
- `< 440px`: huddle also dropped

### Watermark positions (fixed pixels)
- **Huddle**: centred at `CARD_W/2 + 30`, vertically centred
- **Tackle**: x=`-20`, y=`135` (from top)
- **Running**: x=`-55`, bottom edge at `CARD_H - 135`
- **Kicking**: x=`CARD_W - wm_w + 30`, bottom at `CARD_H * 0.04` from bottom
- **Competition TR**: `CARD_W - efnl_w/2 + 30`, rotated 10°, `source-over` at 4–5% opacity

### Watermark sizes (% of CARD_W)
- Huddle: 39%, Tackle: 39%×0.95, Running: 36%×0.95, Kicking: 35%

### Watermark opacity
- `source-over` blend on all cards
- Scales from 30% (dark cards, L=0) to 20% (light cards, L=1.0)
- Formula: `Math.max(0.20, Math.min(0.30, 0.30 - (pL * 0.20)))`

---

## Colour detection

Colours loaded from `assets/colours.json` keyed by club name (e.g. `"Norwood"`, `"South Croydon"`). On fixture upload:
1. Club name detected from `data-testid="team-header-name"` DOM element
2. Age group suffix stripped: `"Beaconsfield Football Club"` → `"Beaconsfield"`
3. Looked up in `colours.json`; falls back to `_default` if not found
4. Gold teams: primary/secondary swapped automatically

### Colour usage
- Card gradient: primary colour dark→light top→bottom
- Our team pill: `lighten(primary, 0.20–0.25)`
- Opposition pill: `lighten(primary, 0.06–0.12)`
- Date badge: secondary colour
- Year bar text: secondary colour
- Bye/finals lines: secondary colour
- Pill text: `bestText(lighten(primary, 0.18))` — contrast-aware black or white

---

## PlayHQ fixture detection

### Title formats handled
- Format A: `"Norwood U12 Purple Fixture for Norwood (Eastern Football Netball League) U12 - B 2026 | PlayHQ"`
- Format B: `"Lysterfield U12 Teal Fixture for Lysterfield U12 - A 2026 | PlayHQ"` (no league in brackets)

### DOM elements used (primary source)
- **`data-testid="team-header-name"`** → full team name (e.g. `"Lysterfield U12 Teal"`)
- **`[class*="sc-1ttrao6-6"]`** → league name (e.g. `"Eastern Football Netball League, Eastern Football Netball League, 2026"`) — first comma segment used

### League name aliases
`EFNL` → `"Eastern Football Netball League"` (and others in `LEAGUE_ALIASES`)

---

## Known quirks & decisions

- **Joint teams** (e.g. `Norwood Gold/Heathmont`): primary club = first part before `/`, colour stripped. Display name: `U12 Gold / Heathmont 2026`. Club name: `Norwood`. Assets load for primary club only. When joint team appears as an *opponent*, `getCrest` strips the secondary team and colour to find the primary club's crest.
- **Slash normalisation**: `shortName()` and `getCrest()` both normalise `/` to ` / ` before processing, so `"Gold/Heathmont"` and `"Gold / Heathmont"` are handled identically.
- **Badge rendering**: pixel-scans offscreen canvas to find content bounding box, strips black padding. `MAX_BW=184, MAX_BH=92`. Vertically centred within `MAX_BH`, left-anchored. Wide logos (Heathmont, Glen Waverley) scale up to 184px wide.
- **Home venue**: stored as plain venue name in the field; `HOME VENUE:` / `HOME VENUES:` prefix added on the card. Venue suppressed from per-row pills when it matches the dominant home venue (≥80% of home games). Multiple venues separated by `/`.
- **Opposition pill fallback**: no crest PNG → first letter of club name on secondary-colour background with `bestText(secondary)` letter colour.
- **Score display**: completed games show `homeScore-awayScore` in VS circle. Win/loss/draw ring: green/red/orange.
- **Beaconsfield**: PlayHQ name includes `"Football Club"` — `shortName()` strips common suffixes including standalone `"Football"`.
- **wm_efnl.png**: palette PNG transparency lost in project folder. Use `.TXT` rename trick for upload. See watermark processing notes above.
- **Loading a new fixture file**: always resets card, clears previous club assets (`state.ourCrest = null`, `state.badgeImg = null`) before loading new ones.
- **No Norwood/EFNL hardcoding**: all variable names, fallbacks and defaults are generic. The `"Norwood"` entry in `DEFAULT_COLOURS` is data, not a default.

---

## Version history

| Version | Key changes |
|---------|-------------|
| 0.114 | Year bar gap reduced; separator line removed |
| 0.113 | Year bar separator line removed |
| 0.112 | Year bar ("2026 FIXTURES") added above header; year stripped from team name |
| 0.111 | getCrest normalises slash before joint team split |
| 0.110 | getCrest used for opponent pill lookup; slash spacing normalised in shortName |
| 0.109 | state.norwood_crest → state.ourCrest; all Norwood/purple hardcoding removed |
| 0.108 | Joint team detection (Norwood Gold/Heathmont); shortName strips Football suffix |
| 0.107 | Home venue field stores plain name; prefix added on card |
| 0.106 | Badge vertical centering within MAX_BH space |
| 0.104 | Badge MAX_BW doubled to 184px for wide logos; watermark opacity slider removed |
| 0.100 | Watermarks simplified to source-over; all blend mode complexity removed |
| 0.98  | detectTeamInfo uses data-testid + sc-1ttrao6-6 as primary DOM sources |
| 0.97  | leagueToSlug handles abbreviations; LEAGUE_ALIASES for full name expansion |
| 0.95  | shortName strips club suffixes; Beaconsfield fix |
| 0.93  | Club suffix stripping for colours/asset lookup |
| 0.92  | detectTeamInfo format B regex for titles without league brackets |
| 0.88  | Loading new fixture resets card and requires Generate Card press |
| 0.84  | Competition-aware assets folder; COL_VS/COL_GAP scale with card width |
| 0.80  | Card width scales with height (A4 proportions, min 560px) |
| 0.75  | Watermarks progressively hidden on short cards |
| 0.70  | Opposition pill letter initial on secondary colour background |
| 0.60  | Watermark blend mode locked; monogram opacity adaptive |
| ~0.40 | Migrated from Python/Playwright HTML to Canvas 2D approach |
| ~0.10 | Original Python static HTML generator with Barlow Condensed font |
