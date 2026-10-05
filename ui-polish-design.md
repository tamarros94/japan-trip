# UI polish design: navigation, quick wins, Tokyo night reskin

Status: draft for review. No site code has been changed yet. Visual reference: `mockup-tokyo-night.html` (repo root).

## Context

The site is one file (`index.html`, about 993 KB, 885 places). Before the social features (reviews, shared trips, profiles) are built, the places browser and the navigation need to be fixed and restyled. Complaints from the owner: cluttered, hard to read, looks dated, does not feel like Japan. Highest priority: UX and UI polish. Four audits (UX, performance, code health, plus a manual check) produced the findings below.

## Decisions already made

| Decision | Choice |
|---|---|
| Focus areas | Places browser and overall navigation |
| Scope of this round | A (quick wins) + B (navigation) + C (visual reskin) |
| Visual direction | Tokyo night (dark indigo app look), from fresh mockups. Only the emoji category icons are a must keep |
| Filter pattern | Bottom sheets for filters and the full category list |
| Bugs | Fix the quoted title bug here. Sync and AI escaping bugs get a separate round before the social features |
| Perf | Include deferring the pdf-lib and Leaflet scripts in round A |
| Single file | Stays single file for this round |

## Round A: quick wins (low risk, mostly CSS)

1. **Tap targets to 44px minimum.** Save heart (`.card-save-btn`, `index.html:158`), itinerary remove and move controls (`:397-399`), nights buttons (`:458`), shopping check circle (`:324`), modal close (`:240`), sign out link (`:315`), nav pills and lodging chips (`:234`, `:283`).
2. **Light theme contrast.** `--border` (about 1.3:1) and `--muted` (about 3.4:1) fail. `--accent`/`--gold` (about 3.9:1) fail for text. Replace with values that reach 4.5:1 for text and 3:1 for borders. Fix `.it-current-card h2` (dark text on a dark gradient, `:408`) and other hardcoded dark only colors.
3. **Focus states.** One global `:focus-visible` ring. Give the `appearance: none` checkboxes a visible focus style.
4. **Real buttons for click handlers on spans.** "What is Tabelog?", read more, select sheet options, lodging cards, itinerary stop names. Keyboard reachable, with a role.
5. **Labels.** `aria-label` on icon only buttons (theme, close, heart, nights, add). A `<label>` on the main search box. `role="dialog"` on the modals, drawer and select sheet.
6. **Theme toggle** moves into the header row so it stops covering the left nav pill (`:50`).
7. **Quoted title bug.** `esc()` (`:2169`) does not escape quotes and is used inside attributes (`:2394`, `:2414`, `:2416`, `:2179`). 3 titles and 7 veg notes contain `"`, so their heart keys are cut off and do not match on My Area. Add an attribute safe escape, or set `data-*` through the DOM. Saved keys already stored for those 3 titles are cut off and will not match (minor, accepted).
8. **Script loading.** `leaflet.js` and `pdf-lib.min.js` (`:1457-1458`) block parsing. Load pdf-lib only when "export PDF" is used. Defer Leaflet. **Check first** that no top level code calls `L.` before the maps are opened; maps are created lazily today, so this is expected to be safe.
9. **Minimum label size** 12px (many pixel font labels are about 10px).

## Round B: navigation

Problem: pages are `display` toggles done with inline onclick (`:615`, `:1141`, `:1193`, `:1393-1399`, `:2241`, `:2250`, `:4742`). There is no history, so the phone's back button leaves the site, scroll is not reset or restored, and the back buttons return to the wrong place.

Design:
* One `showPage(name)` helper that opens or closes `shoppingPage`, `lodgingPage`, `itineraryPage`, `myAreaPage`, runs their init (`lodgingInit`, `renderMyArea`), resets scroll on open, and restores the list scroll on return.
* `history.pushState` on open and a `popstate` handler. Every back button calls `history.back()`, so My Area then itinerary then back returns to My Area.
* Modals, the drawer and the select sheet close on Escape, and return focus to the opener.
* The hash router is the base the social links (`#trip=`, `#u=`) will reuse, so it is designed for arbitrary routes, not just these four pages.
* Signed out behavior is unchanged.

Navigation structure (from the mockup):
* **Mobile:** bottom tab bar with 6 items: מקומות (home, active marker), קניות, לינה, מסלולים, האזור שלי, עוד. The "עוד" sheet holds מפות, מי אני, תכנון, פידבק. Note "מפות" opens the maps drawer today, it is not the home list.
* **Desktop (900px and up):** top nav with all 8 items.
* Page destinations and info items are visually separated.

## Round C: visual reskin (Tokyo night)

Reskin by replacing design tokens first, then restyling components.

**Tokens (from the mockup, dark / light):**
* Background `#0e0f23` / `#f3f3fb`; surface `#181a38` / `#ffffff`; raised surface `#22254a` / `#eaeaf8`; line `#2f3360` / `#d2d3ea`
* Text `#eeeeff` / `#1a1b3a`; muted `#aeb1d6` / `#515479`; primary `#9b9cff` / `#4338ca`
* One accent per category, used only as a glow ring and selected chip border, never as text
* Font: Heebo only. Body 16px, labels 12px minimum, card title 18px

**Components:** header with sign in and theme toggle; search box; horizontally scrolling category chips with a trailing "עוד קטגוריות" chip; toolbar (sort, filters with active count badge, AI search toggle); bottom sheets with a dimmed backdrop; place cards with a clear order: emoji ring and title, one meta line (category and city), grouped badges, note, recommender, maps button; personal pick card with a gold flag.

**Filters (behind the sheet):** must book ahead, verified rating only, personal picks only, vegetarian/vegan only, recommender, city, price range. Sort options: regular, score, price.

**Phasing:** C1 covers tokens, header, nav, search, chips, filters, cards (the places browser). C2 restyles shopping, lodging, itinerary and My Area using the same tokens. The pixel fonts (Pixelify Sans, DotGothic16) can be dropped only after C2, which also saves font downloads.

## Not in this round

* Cloud sync fixes: a save can overwrite the cloud copy with local state, checklist changes do not sync, a cloud itinerary appears only after reload, account data can leak on a shared device. Separate round, before the social features.
* AI response and URL escaping (`:5328`, `:5147`, `:2405`).
* Deployed worker source is missing from the repo (needed before any worker change).
* Larger performance work: lazy loading the 534 KB places JSON, moving the itinerary and lodging constants out of the main script, batching `render()` with a document fragment, caching `esc()`.
* Stale files in the repo root (`.bak`, mockups), to clean up or ignore.
* The social features themselves (see the earlier plan).

## Known gaps in the mockup

* Emoji rendering, glow rings and contrast were not verified (the render environment had no emoji font, contrast was estimated by hand). Check on a real device.
* The light theme was rendered but tap targets were measured only in the dark theme.
* The 900px desktop layout and the cards grid have not been reviewed closely.
* Emoji for 16 of the 27 categories are placeholders; use the real `CATEGORY_EMOJI` map.

## Build order (each a small PR)

1. Round A (one PR, or split: contrast and focus, tap targets and labels, quote bug, script loading).
2. Round B (`showPage`, history, Escape handling).
3. Round C1 (tokens and places browser), then C2.

## Verification

* Headless screenshots at 390px and 900px, both themes, for the places browser, each page, and each sheet.
* Script that lists interactive elements smaller than 44px and checks for horizontal overflow (already used on the mockup).
* Contrast check of the token pairs in both themes.
* Manual on a phone: open shopping, then back returns to the list at the same scroll; My Area to itinerary and back returns to My Area; Escape closes modals.
* Save a heart on each of the 3 quoted titles and confirm they appear in My Area.
* Regression: signed out network tab shows no new calls; checklist, saved recs and itinerary still sync between two devices for a signed in user.
