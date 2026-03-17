# Circuitmatic — Industrial 2 Migration & Launch Implementation Plan

## Context for Claude Code

You are working on **Circuitmatic**, a single-file PWA (`index.html`) that is an NEC electrical code reference calculator for licensed electricians. The app currently has **26 horizontal-scroll tabs** containing calculators, reference charts, field tools, and utilities — all in one ~16,000-line HTML file.

The goal is to migrate the navigation architecture to a **hub-and-spoke model** using the "Industrial 2" design shell (`circuitmatic-industrial2.html`) as the new UI foundation. The calculator logic, data tables, and all existing functionality must be preserved — only the **navigation layer and visual styling** change.

**Developer**: Dominic Defrancesco
**Platforms**: iOS (App Store) + Android (Google Play) via native wrapper (Capacitor/PWABuilder)
**Code editions**: NEC 2020, 2023, 2026
**Domain**: circuitmatic.app
**Revenue model**: Freemium — 3 tiers (Apprentice free / Journeyman / Foreman), monthly subscription. Pricing TBD.
**Distribution**: 100% offline PWA wrapped for app stores. No servers, no backend, no database.

---

## Architecture Overview

### Current State (index.html)
- 26 tabs in a horizontal scroll bar (`tab-bar` with `tab-btn` buttons)
- `switchTab(id, btn)` function toggles `tab-panel` visibility
- Each tab is a `<div class="tab-panel" id="tab-{id}">`
- Single flat navigation — every tool is 1 tap + scroll to find
- Fonts: Inter, Roboto Mono, Orbitron
- Color tokens: `--bg: #080a10` dark navy theme

### Target State (Industrial 2)
- **5 bottom tab bar screens**: Home, Calculators, Reference, Projects, Settings
- **6 category bottom sheets** under Calculators that slide up with tool lists
- Hub-and-spoke: Home screen has quick launch, search, recents, category tiles
- Fonts: Plus Jakarta Sans, DM Mono (delete Inter, Roboto Mono, Orbitron)
- Color tokens: `--bg: #111213` deep space theme from industrial2.html

---

## Subscription Tiers & Feature Gating

### IMPORTANT: Tier splits and pricing are TENTATIVE. Do not hardcode specific feature-to-tier assignments as final. Build the gating system to be easily reconfigurable — a single config object that the developer can adjust before launch.

### Tier Structure (DRAFT — subject to change)

| | Apprentice (Free) | Journeyman (~$3.99/mo TBD) | Foreman (~$7.99/mo TBD) |
|---|---|---|---|
| **Trial** | 3-7 day full access (length TBD), then drops to free tier | — | — |
| **Core Calcs** | A subset of calculators (likely V.Drop, Ampacity, Box Fill, Conduit Fill basic) | Most or all calculators | All calculators |
| **Reference** | A subset of reference tables | Most or all reference tables | All reference tables |
| **Field Tools** | Basic utilities (Wire Colors, Fractions, DIP Switch) | + Projects, PDF Scanner, Code Quiz | Everything |
| **Exports** | Limited (screenshot only) | PDF, SMS, Email, AirDrop | + Branded exports, company logo, letterhead |
| **Power/Advanced** | Basic calcs only | + Transformer, Solar/PV, EV Charger, etc. | + Arc Flash, Fault Current |
| **Specialty** | — | — | AI Field Assistant (future), Company branding |
| **Code Year** | TBD (may be current year only, or all) | All years (2020/2023/2026) | All years |

### Feature Gating Implementation

Build a configurable gate system. The developer will finalize which features go in which tier before launch.

```javascript
// Tier constants
var TIER_APPRENTICE = 0;
var TIER_JOURNEYMAN = 1;
var TIER_FOREMAN = 2;

// Feature registry — EASILY EDITABLE config object
// Developer will adjust these assignments before launch
var FEATURE_GATES = {
  // === Calculators ===
  'calc_vd': TIER_APPRENTICE,
  'calc_amp': TIER_APPRENTICE,
  'calc_bf': TIER_APPRENTICE,
  'calc_cf_basic': TIER_APPRENTICE,
  'calc_mc': TIER_APPRENTICE,      // may move to Journeyman
  'calc_svc': TIER_APPRENTICE,     // may move to Journeyman
  'calc_dr': TIER_JOURNEYMAN,
  'calc_rc': TIER_JOURNEYMAN,
  'calc_cl': TIER_JOURNEYMAN,
  'calc_feeder': TIER_JOURNEYMAN,
  'calc_cf_data': TIER_JOURNEYMAN,
  'calc_cf_bends': TIER_JOURNEYMAN,
  'calc_gnd_full': TIER_JOURNEYMAN,
  'calc_xfmr': TIER_JOURNEYMAN,
  'calc_pv': TIER_JOURNEYMAN,
  'calc_ev': TIER_JOURNEYMAN,
  'calc_gs': TIER_JOURNEYMAN,
  'calc_pn': TIER_JOURNEYMAN,
  'calc_af': TIER_FOREMAN,
  'calc_fc': TIER_FOREMAN,
  // === Exports ===
  'export_pdf': TIER_JOURNEYMAN,
  'export_sms': TIER_JOURNEYMAN,
  'export_email': TIER_JOURNEYMAN,
  'export_branded': TIER_FOREMAN,
  // === Field tools ===
  'tool_projects': TIER_JOURNEYMAN,
  'tool_scanner': TIER_JOURNEYMAN,
  'tool_quiz': TIER_JOURNEYMAN,
  // === Specialty ===
  'ai_assistant': TIER_FOREMAN,
  'company_branding': TIER_FOREMAN,
};

function getUserTier() {
  var sub = JSON.parse(localStorage.getItem('cm_subscription') || '{}');
  if (sub.trialStart && !sub.paid) {
    var trialDays = sub.trialDays || 5;
    var elapsed = Date.now() - sub.trialStart;
    if (elapsed < trialDays * 24 * 60 * 60 * 1000) return TIER_FOREMAN;
  }
  return sub.tier || TIER_APPRENTICE;
}

function canAccess(featureKey) {
  return getUserTier() >= (FEATURE_GATES[featureKey] || TIER_FOREMAN);
}

function requireTier(featureKey, callback) {
  if (canAccess(featureKey)) {
    callback();
  } else {
    showUpgradeSheet(featureKey);
  }
}
```

### Upgrade Prompt Sheet
When a locked feature is tapped, show a bottom sheet (same pattern as category sheets):
- Feature name + one-line description
- Which tier unlocks it (badge)
- "Start Free Trial" button (if unused)
- "Upgrade" button → subscription flow
- "Maybe Later" dismiss
- Blurred/dimmed preview of locked calculator behind the sheet

### Visual Gating in Navigation
Locked features in category sheets and tool lists:
- Dimmed row (opacity: 0.6)
- Lock icon replacing chevron
- Tier badge pill (using existing `b-pro` / `tb-pro` classes)
- Tap opens upgrade sheet, not dead tap
- **Locked tools are VISIBLE, never hidden**

### Trial System
```javascript
function startTrial() {
  var sub = JSON.parse(localStorage.getItem('cm_subscription') || '{}');
  if (!sub.trialStart) {
    sub.trialStart = Date.now();
    sub.trialDays = 5; // configurable
    localStorage.setItem('cm_subscription', JSON.stringify(sub));
  }
}

function checkTrialExpiry() {
  var sub = JSON.parse(localStorage.getItem('cm_subscription') || '{}');
  if (sub.trialStart && !sub.paid) {
    var trialDays = sub.trialDays || 5;
    if (Date.now() - sub.trialStart >= trialDays * 24 * 60 * 60 * 1000) {
      sub.tier = TIER_APPRENTICE;
      sub.trialExpired = true;
      localStorage.setItem('cm_subscription', JSON.stringify(sub));
    }
  }
}
```

Call `checkTrialExpiry()` on app init. Expiry is graceful — gate on next launch, not mid-session.

---

## Onboarding Tutorial

### Triggers
- First launch (no `cm_onboarding_complete` in localStorage)
- Settings → "Replay Tutorial"

### Flow (5 screens, swipe/tap)

1. **Welcome** — Wordmark + circuit trace animation. "Built by electricians, for electricians."
2. **The Hub** — Callout bubbles on Quick Launch, Search, Recents, Categories.
3. **Smart Calculators** — V.Drop result with pass/fail, NEC article refs, code year toggle.
4. **Field Tools** — Projects, scanner, quiz, punch list. "Your field office in your pocket."
5. **Your Trial** — 3 tier cards. "Start Exploring" CTA → starts trial. Small "use free tier" link.

### UI: Full-screen overlay, `--bg` background, progress dots, skip button on every screen, swipe + next button. Circuit trace animation style (gold on dark). On dismiss: set `cm_onboarding_complete`, call `startTrial()`.

---

## Phase 1: Navigation Shell

### 1.1 — Bottom Tab Bar
Replace horizontal `tab-bar` with fixed bottom nav:
```
Home | Calculators | Reference | Projects | Settings
```
- 5 top-level `<div class="screen" id="screen-{name}">`
- One visible at a time
- `position: fixed; bottom: 0` with `padding-bottom: env(safe-area-inset-bottom)`
- Active tab: gold accent via `class="active"`
- Template: industrial2.html `<nav class="tabbar">`

### 1.2 — Home Screen (screen-home)
1. **Header**: CIRCUITMATIC wordmark (Plus Jakarta 800), NEC year toggle, dark/light pill, profile icon
2. **Hero cockpit**: "Built for the field", stat counters, NEC badge with pulse dot
3. **Quick Launch** (6): V.Drop, Box Fill, Ampacity, Conduit Fill, Motor, Panel Sched
4. **Global search**: High-contrast, queries calcs + tables + NEC sections
5. **Recents**: Last 5 used (localStorage `cm_recents`), colored chips, 1-tap return
6. **Category tiles**: 6 horizontal-scroll cards → open category sheets
7. **All tools list**: Every calculator, sorted by popularity
8. **Quick tools pills**: Wire Colors, Fractions, DIP Switch, Resistor
9. **AI strip**: "Coming soon" + Notify CTA

### 1.3 — Category Bottom Sheets
From industrial2.html overlay/sheet pattern: backdrop, slide-up sheet, drag handle, header, item list with icon/name/subtitle/badge/chevron/lock.

---

## Phase 2: Calculator Tab Migration Map

### Conductors & Load (6 tools)
| Tab ID | Name | Lines |
|---|---|---|
| `tab-vd` | Voltage Drop | ~140 |
| `tab-amp` | Ampacity | ~135 |
| `tab-dr` | Derate | ~77 |
| `tab-rc` | Receptacle Spacing | ~75 |
| `tab-cl` | Commercial Load | ~137 |
| `tab-feeder` | Feeder Sizing | ~8854 |

### Conduit & Raceway (5 tools)
| Tab ID | Name | Lines | Notes |
|---|---|---|---|
| `tab-cf` | Conduit Fill | ~532 | Includes power + data sub-modes |
| `tab-bf` | Box Fill | ~147 | **MOVED from Protection** |
| — | Conduit Bends | — | Sub-tab of cf or standalone TBD |
| — | Run Planner | — | May be new or extracted |

### Grounding & Bonding (1 entry, 6 internal sub-tabs)
| Tab ID | Sub-tabs |
|---|---|
| `tab-gnd` | EGC, GEC, SBJ, SSBJ, EBJ, Bonding (via `gndSetSub()`) |

### Power Systems (5 tools)
| Tab ID | Name | Lines | Badge |
|---|---|---|---|
| `tab-mc` | Motor Circuit | ~118 | — |
| `tab-xfmr` | Transformer | ~233 | — |
| `tab-pv` | Solar / PV | ~89 | NEW |
| `tab-af` | Arc Flash | ~256 | PRO |
| `tab-fc` | Fault Current | ~132 | PRO |

### Service & Panels (4 tools)
| Tab ID | Name | Lines | Badge |
|---|---|---|---|
| `tab-svc` | Dwelling Service | ~74 | — |
| `tab-pn` | Panel Schedule | ~77+ | — |
| `tab-gs` | Generator Sizing | ~152 | — |
| `tab-ev` | EV Charger | ~71 | NEW |

---

## Phase 3: Reference Screen (screen-ref)

Replaces `tab-charts`. Accordion sections (existing `cht-card`/`toggleCht()` pattern), all collapsed by default:

1. **Conductors & Wiring** (7): Cu/Al ampacity, insulation types, THHN, temp correction, bundling, VD table
2. **Conduit & Raceways** (5): EMT fill, RMC/IMC fill, box fill, support spacing, burial depth
3. **Protection & Clearances** (7): OCPD, working space, NEMA, GFCI (dwelling/commercial/appliances)
4. **Wet & Outdoor** (5): Location types, wiring methods, devices, fixtures, common mistakes
5. **Grounding & Bonding** (2): EGC table, GEC table
6. **Motors** (2): 3-ph FLA, 1-ph FLA
7. **Low Voltage & Telecom** (2): RJ-45 pinout, fiber color code
8. **Fire Alarm** (2): Detector spacing, Art 760
9. **Reference & Math** (3): Torque, residential circuits, electrical math

---

## Phase 4: Projects Screen (screen-projects)

1. **Projects manager** — Port `tab-notes`
2. **PDF Scanner** — Port `tab-scan`
3. **Code Quiz** — Port `tab-quiz`
4. **Quick Tools** (inline): Wire Colors (`tab-wc`), Fractions (`tab-fr`), DIP Switch (`tab-dip`), Resistor Color Code (new)

---

## Phase 5: Settings Screen (screen-settings)

- NEC code year default (2020/2023/2026)
- Dark/light theme
- Subscription tier + manage link
- Replay tutorial
- About / Terms / Feedback / Version
- AI assistant notification pref
- Support: support@circuitmatic.app

---

## Phase 6: Design System Migration

### 6.1 — Fonts
**Delete**: Inter, Roboto Mono, Orbitron
**Add**: Plus Jakarta Sans (400-800), DM Mono (400-500)
- All `'Inter'` → `'Plus Jakarta Sans'`
- All `'Roboto Mono'` → `'DM Mono'`
- All `'Orbitron'` → `'Plus Jakarta Sans'` weight 800

### 6.2 — Color Tokens
Replace old tokens with industrial2 set:
```css
--bg:#111213; --bg2:#181a1b; --bg3:#1e2022;
--surface:#222426; --raise:#2a2d2f;
--border:rgba(255,255,255,0.07); --border2:rgba(255,255,255,0.11); --border3:rgba(255,255,255,0.18);
--ink:#f0f0ee; --ink2:#c8c8c4; --ink3:#888884; --ink4:#555552;
--gold:#ffc400; --gold-lt:#ffe566; --gold-dk:#cc9900;
--ib:#1e2d4a; --ib-s:#6699ff;
--ic:#0e2d35; --ic-s:#33ccdd;
--it:#0e2d22; --it-s:#33cc88;
--ia:#2d1e0e; --ia-s:#ff9944;
--ir:#2d0e18; --ir-s:#ff5577;
--is:#242424; --is-s:#aaaaaa;
--ig:#0e2d1a; --ig-s:#44dd88;
```

### 6.3 — Input Restyling
```css
input.fi, select.fi {
  font-family: 'DM Mono', monospace;
  font-size: 13px;
  color: var(--ink);
  background: var(--bg2);
  border: 1px solid var(--border2);
  border-radius: 8px;
  padding: 10px 12px;
  box-shadow: inset 0 2px 4px rgba(0,0,0,.35);
}
input.fi:focus, select.fi:focus {
  border-color: var(--gold);
  background: var(--bg3);
  box-shadow: inset 0 2px 4px rgba(0,0,0,.35), 0 0 0 2px rgba(255,196,0,.15);
}
```

### 6.4 — Light Mode
Standardize on `body.light` (not `body.light-mode`). Port industrial2 light overrides.

---

## Phase 7: PDF Auto-Naming

```javascript
function autoFilename(calcType, details) {
  var d = new Date();
  var stamp = d.getFullYear()+'-'+String(d.getMonth()+1).padStart(2,'0')+'-'+String(d.getDate()).padStart(2,'0');
  var safe = (details||'').replace(/[^a-zA-Z0-9_-]/g,'_').substring(0,40);
  return 'Circuitmatic_'+calcType+(safe?'_'+safe:'')+'_'+stamp+'.pdf';
}
```

Replace all `prompt()` filename dialogs and hardcoded `.save('...')` calls.

---

## Phase 8: Search Enhancement

Home screen search bar queries calcs + tables + NEC sections. Fuzzy match electrician shorthand.

```javascript
var searchIndex = [
  { type:'calc', id:'vd', name:'Voltage Drop', keywords:'vdrop wire size distance conductor ch9', cat:'conductors' },
  { type:'calc', id:'amp', name:'Ampacity Lookup', keywords:'ampacity amps 310.16 wire rating', cat:'conductors' },
  { type:'chart', id:'cht-ampacity', name:'Conductor Ampacity — Copper', keywords:'ampacity copper 60 75 90 310.16', cat:'ref' },
  // ... all tools and charts
];
```

---

## App Store & Launch Context

### Costs
- Apple Developer: $99/year · Google Play: $25 one-time · Netlify: ~$5/mo · Domain: ~$10-15
- Break-even: ~25 downloads

### App Store Screenshots (6-8, in order)
1. "Voltage Drop in Seconds"
2. "2020 / 2023 / 2026 Code — Your Code, Your State"
3. "Residential Service Sizing"
4. "Conduit Bending Made Easy"
5. "Conduit Fill — Any Raceway"
6. "GFCI / AFCI — Instant Lookup"
7. "Motor Circuit Sizing"
8. "Built for the Field — 100% Offline"

### QA (from launch plan)
- 5 test cases per calculator: normal, edge low, edge high, boundary, NEC gotcha
- Source of truth: physical code books (2020, 2023, 2026)
- Cross-reference competing apps
- **All 3 code years must produce different results where NEC changed**

### Front-End Polish Items (from launch plan, TBD)
- Pipe bends visual polish / 3D model views
- Verify 2020/2023/2026 differences especially new NEC modifications
- Labor tracker (considering)
- PDF output polish
- Electrical bot / AI assistant (future)

---

## Critical Warnings

1. **DO NOT use "E" for voltage.** "V" is field convention. Logo "E" is brand styling only.
2. **NO floating action buttons.** Use inline tiles or header buttons. No Material Design FABs.
3. **Preserve iOS 16px zoom fix:** `input:focus { font-size: 16px !important; }` in mobile media query.
4. **Preserve safe area insets.** `env(safe-area-inset-bottom)` and `env(safe-area-inset-top)`.
5. **Do not rename localStorage keys.** Existing user data persists. New keys use `cm_` prefix.
6. **`setCodeYear()` is globally reactive.** Must remain accessible from wherever the toggle lives.
7. **Feeder tab = 8,854 lines.** Do not refactor internals. Treat as sealed module.
8. **Preserve all tab panel content byte-for-byte.** Migration changes navigation, not calculator logic.
9. **FEATURE_GATES config is NOT final.** Build for easy reconfiguration.
10. **New features may be added.** Architecture must accommodate additions.
11. **Trial expiry = graceful.** Gate on next app launch, not mid-calculation.
12. **Locked features = visible.** Never hidden. Lock icon + tier badge + upgrade sheet on tap.
13. **Onboarding = required but skippable.** Skip button on every screen.

---

## Implementation Order

### Block A — Visual Foundation (safe to do during QAQC)
1. Font migration
2. Color token migration
3. Input restyling

### Block B — Navigation Architecture
4. Navigation shell (5 screens + bottom tab bar)
5. Category sheets (6 overlays)
6. Move tab panels into new screens
7. Home screen build
8. Reference screen build
9. Projects screen build
10. Settings screen build

### Block C — Polish
11. PDF auto-naming
12. Search enhancement

### Block D — Monetization (parallel or after)
13. Feature gate system
14. Trial system
15. Visual gating (lock icons, badges)
16. Onboarding tutorial
17. Upgrade sheet UI
18. Subscription flow (RevenueCat / App Store IAP)

### Block E — Launch
19. Full QA pass
20. App store screenshots
21. Native wrapper (Capacitor/PWABuilder)
22. Submit to stores

---

## Files Reference

- `index.html` (`index__9_.html`) — 15,948-line PWA. All calculator logic.
- `circuitmatic-industrial2.html` — 927-line nav shell / design reference.
- `circuitmatic_ia_plan_v2_final.svg` — Visual IA map.
- `Circuitmatic_Launch_Plan.txt` — Business strategy, pricing, QA, app store listing.
