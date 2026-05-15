# TM22 Ballistic Logger — Bolt Build Guide

## What You're Building

A 22LR competition ballistic calculator Android app. It has a custom drag model (TM22) that's more accurate than anything on the market for rimfire. The app connects to Bluetooth devices (wind meter, rangefinder, weather station, DOPE display card) and provides real-time firing solutions for NRL22/PRS Rimfire matches.

## Architecture

- **Framework:** Vite + React + Tailwind CSS
- **Mobile:** Capacitor wrapping the web app into an Android APK
- **BLE:** @capacitor-community/bluetooth-le plugin
- **Data:** localStorage for persistence (no backend, no database)
- **Single-page app** with tab navigation: HUD, Truing, Profiles, BLE, Guide

## Design System

- Dark theme: bg `#0f172a`, cards `#1e293b`, borders `#334155`, text `#f1f5f9`
- Accent: `#ea580c` (orange)
- Status colors: green `#22c55e`, yellow `#eab308`, red `#ef4444`, blue `#3b82f6`
- Font: system-ui
- Mobile-first: max-width 480px, centered
- Bottom tab bar: fixed position, icon + label per tab
- Cards: rounded corners (8px), subtle border, consistent padding (12px)
- Inputs: dark background matching page bg, light border, 14px font
- Buttons: orange accent, white text, 44px minimum touch height
- Labels: uppercase, 11px, muted color, 0.5 letter-spacing

---

## WORK GROUP 1: Project Scaffold + Ballistic Engine

### What this does
Sets up the Vite + React + Tailwind project and implements the core physics engine. No UI yet — just the math that everything else depends on.

### Prompt for Bolt

> Create a new Vite + React + Tailwind project. Set up the project structure with a single App.jsx component.
>
> Create a file called `src/engine/ballistics.js` that exports the following functions. This is a point-mass ballistic trajectory solver for 22LR ammunition.
>
> **Constants:**
> - STD_SPEED_OF_SOUND = 1116.4 fps
> - GRAVITY = 386.4 in/s²
> - DRAG_C_STD = (0.5 × 0.002378 × π × 32.174) / (4 × 144)
>
> **Data tables (paste these exactly):**
>
> G1_TABLE — standard G1 drag coefficient vs Mach array:
> ```
> [[0,0.2629],[0.05,0.2558],[0.1,0.2487],[0.15,0.2413],[0.2,0.2344],[0.25,0.2278],[0.3,0.2214],[0.35,0.2155],[0.4,0.2104],[0.45,0.2061],[0.5,0.2032],[0.55,0.202],[0.6,0.2034],[0.65,0.2165],[0.7,0.223],[0.725,0.2313],[0.75,0.2417],[0.775,0.2546],[0.8,0.2706],[0.825,0.2901],[0.85,0.3136],[0.875,0.3415],[0.9,0.3734],[0.925,0.4084],[0.95,0.4448],[0.975,0.4805],[1,0.5067],[1.025,0.5191],[1.05,0.52],[1.075,0.515],[1.1,0.5063],[1.125,0.495],[1.15,0.483],[1.2,0.459],[1.25,0.437],[1.3,0.418],[1.35,0.401],[1.4,0.386],[1.45,0.373],[1.5,0.361],[1.55,0.35],[1.6,0.34]]
> ```
>
> TM22_TABLE — custom drag model for 22LR, derived from Applied Ballistics Doppler lab data (SK Match ammunition). This is the core innovation — a drag curve measured specifically for round-nose heeled 22LR bullets:
> ```
> [[0.500,0.0638],[0.550,0.0638],[0.600,0.0638],[0.625,0.0647],[0.650,0.0656],[0.675,0.0662],[0.700,0.0664],[0.725,0.0665],[0.750,0.0668],[0.775,0.0672],[0.800,0.0682],[0.825,0.0696],[0.850,0.0716],[0.875,0.0743],[0.900,0.0829],[0.925,0.0940],[0.950,0.1169],[0.975,0.1713],[1.000,0.2188]]
> ```
>
> **Functions to implement:**
>
> `lookupCd(table, mach)` — Linear interpolation in a Cd vs Mach table. Clamp to table edges.
>
> `calculateDrop(params, cdTable)` — Point-mass trajectory solver. Takes: { distance (yards), muzzleVelocity (fps), bc, zeroRange (yards), scopeHeight (inches), temperature (°F), densityAltitude (ft) }. Returns: { dropMils, dropMOA, velocity, mach, tof, dropInches }.
> - Step size: 0.5 yard increments
> - Air density ratio: rhoRatio = exp(-densityAltitude / 34000)
> - Speed of sound adjustment: sos = 1116.4 × sqrt((temperature + 459.67) / 518.67)
> - At each step: compute Mach, lookup Cd from table, compute drag deceleration, update velocity and position
> - At zeroRange: compute sight angle that zeros the rifle
> - At target distance: compute drop in mils and MOA relative to sight line
>
> `solveBcFromDialUp(dialUpMils, distance, zeroRange, scopeHeight, mv, temp, da, cdTable)` — Binary search solver. Given an actual dial-up at a known distance, solve for the BC that produces that exact dial-up. Returns { bc, clamped }. Search range: 0.01 to 0.5 for TM22, 0.01 to 0.8 for G1. Tolerance: 0.001 mils. Max iterations: 50.
>
> `calcVelocityAtDist(distance, mv, bc, cdTable)` — Return terminal velocity at a given distance.
>
> `lookupSteppedBc(distance, truingPoints)` — Given an array of truing points (each with distance and bc), interpolate BC at any distance. Below the first point: use first BC. Above last: use last BC. Between: linear interpolation by distance.
>
> `getConfZone(distance, zeroDistance)` — Returns "low" if distance < 2.5 × zeroDistance, otherwise "high". Used to flag unreliable short-range data.
>
> **Wind functions:**
>
> `calcWindDrift(distance, mv, bc, cdTable, crosswindMph)` — Lag-time (Didion) method: drift = crosswind_fps × (TOF - distance/MV). Returns drift in mils.
>
> `calcCrossingAngle(targetBearingDeg, windFromDeg)` — Returns crossing angle. crosswind = windSpeed × sin(relativeAngle).

### Test it works
After building, verify in browser console:
- `calculateDrop({distance:200, muzzleVelocity:1056, bc:0.135, zeroRange:50, scopeHeight:2.6, temperature:59, densityAltitude:0}, TM22_TABLE)` should return ~7.3 mils drop.

---

## WORK GROUP 2: Data Models + Profiles Tab

### What this does
Creates the data structures for rifles and ammo, the localStorage persistence layer, and the Profiles tab UI where users create/edit their rifle and ammo setups.

### Prompt for Bolt

> Create a Profiles tab with two sections: Rifles and Ammo.
>
> **Rifle profile fields:**
> - name (text)
> - zeroDistance (number, default 50, in yards)
> - scopeHeight (number, in inches — CRITICAL: this is measured center-of-bore to center-of-scope)
> - barrelLength (number, default 20, in inches)
> - twistRate (number, default 16)
> - turretUnit (dropdown: MIL or MOA, default MIL)
> - trackingCorrection (number, default 1.0 — 1.0 = 100% tracking)
>
> **Ammo profile fields:**
> - profileName (text, optional display name)
> - manufacturer (dropdown: SK, Lapua, Eley, CCI, Federal, RWS, Other)
> - product (dropdown: changes based on manufacturer selection)
>   - SK: Standard Plus, Rifle Match, Long Range Match, Flatnose
>   - Lapua: Center-X, Midas+, X-Act, Polar Biathlon, Club
>   - Eley: Tenex, Match, Club, Contact, Force
>   - CCI: Standard Velocity, Green Tag, Pistol Match, Blazer
>   - Federal: Gold Medal, Champion, Auto Match, Ultra Match
>   - RWS: R50, R100, Rifle Match, Target Rifle
>   - Other: Custom
> - lotNumber (text)
> - muzzleVelocity (number, fps)
> - sd (number, fps)
> - es (number, fps)
> - bulletWeight (number, default 40, grains)
> - baseBc (number, default 0.135)
> - groupSize (number), groupUnit (dropdown: MOA, MIL, IN)
>
> **Persistence:** Save all data (rifles, ammo, truing data, active tab) to localStorage key "tm22_data_v3" as JSON. Load on mount, save on every state change.
>
> **UI pattern:** Each section is collapsible. "Add Rifle" / "Add Ammo" buttons expand an inline form. Saved items show as cards with edit and delete buttons. Delete requires double-tap confirmation (first tap shows "Confirm?", auto-resets after 3 seconds).
>
> Use the dark theme described above. All inputs on dark backgrounds with light borders.

---

## WORK GROUP 3: Truing Tab

### What this does
The Truing tab is where users calibrate the solver against real range data. They shoot at known distances, enter the actual dial-up, and the solver reverse-engineers the BC at each distance.

### Prompt for Bolt

> Create a Truing tab with these sections:
>
> **Rifle + Ammo selector** at the top — two dropdowns that filter which truing dataset you're working with. Truing data is stored per rifle+ammo combination (key: "{rifleId}_{ammoId}").
>
> **Conditions bar:** Three inline inputs — Temp (°F), DA (ft), MV override (fps, placeholder shows ammo profile MV).
>
> **Scope Tracking Test (collapsible `<details>`):**
> - Distance, Dialed amount + unit (MIL/MOA), Measured amount + unit (MIL/MOA/IN)
> - Calculates: expected movement in inches = (dialed × distance × 0.036) for MIL or (dialed / 95.5 × distance) for MOA
> - Tracking % = measured / expected × 100
> - Color: green 98-102%, yellow 95-105%, red outside
> - "Save" button stores trackingCorrection on the rifle profile
>
> **Single Point Entry:**
> - Distance (yards), Dial-up (number) + unit picker (MIL/MOA/IN)
> - Shots (default 10), Group size + unit (MOA/MIL/IN)
> - "Add" button calls solveBcFromDialUp using TM22_TABLE
> - Tracking correction applied: corrMils = dialMils / trackingCorrection
> - Points below 2.5 × zeroDistance flagged as lowConf
>
> **Collection Mode (toggle button):**
> - Grid view: 10 preset distances (75, 100, 125, 150, 175, 200, 225, 250, 275, 302)
> - Each row: distance (editable), dial-up input, shots, group size
> - "Save All" button batch-processes all rows with dial-ups entered
> - Session note field
> - All points share a sessionId (timestamp)
>
> **Truing Points Display:**
> - Cards showing: distance, dial-up, solved BC, TM22 factor (= solvedBc / baseBc), terminal Mach, velocity
> - Green dot if high confidence, yellow if lowConf
> - Edit (✎) and delete (✕) buttons on each card
> - "Clear All" with double-tap confirmation
>
> **Each truing point stores a full snapshot:**
> ```js
> {
>   id, timestamp, sessionId, distance, rawDial, dialUnit, dialMils, corrMils,
>   bc, factor, lowConf, velocity, mach,
>   shots, grpSize, grpUnit,
>   conditions: { temp, da, mv, profileMV, trackingCorr, zeroDistance, scopeHeight, bcModel: "TM22", baseBc }
> }
> ```

---

## WORK GROUP 4: HUD Tab

### What this does
The main firing solution display. Shows elevation and wind holds for the selected distance. This is what the shooter looks at during a match.

### Prompt for Bolt

> Create the HUD tab — the primary display. Layout from top to bottom:
>
> **Rifle/Ammo selector:** Two dropdowns, auto-select first if only one exists.
>
> **Main display card:**
> - Distance input (large, centered) with ▲/▼ buttons for quick increment
> - Increment selector: 1, 5, 10, 25 yard steps
> - Large elevation number (the primary output — e.g., "7.30" in MIL or MOA based on rifle's turretUnit)
> - ▲ UP indicator
> - Badge: "UNTRUED" (yellow) if no truing data, or "TRUED ×N" (green) showing number of truing points
>
> **Flight data row:** Terminal velocity (fps), energy (ft-lbs), TOF (seconds), terminal Mach
>
> **Environment section (collapsible):**
> - MV override (fps), Temp (°F), DA (ft)
> - If BLE weather is connected, show green dot next to "ENVIRONMENT" label and auto-fill temp/DA
> - Show current MV source: "Chrono'd 1065 fps" or "Profile MV 1056 fps"
> - If BLE wind data available, show wind speed and direction
>
> **Wind section (collapsible):**
> - Target Bearing (degrees or clock format like "3:15")
> - Wind FROM direction (degrees or clock)
> - Clock/Degrees toggle
> - Wind 1 speed (mph), Wind 2 speed (mph) — for wind bracket
> - Computed crossing angle display
> - Windage output: W Min and W Max columns in the range table
>
> **Range Table (collapsible):**
> - Min/Max/Step dropdowns (defaults 25/325/25)
> - Grid: Distance | Elevation | W Min | W Max (wind columns only show if wind entered)
> - Current selected distance highlighted in accent color
> - Confidence color on distance numbers (green = trued zone, yellow = outside)
>
> **Solver logic:**
> - If truing data exists: use lookupSteppedBc to interpolate BC, solve with TM22_TABLE
> - If no truing data: use ammo's baseBc with G1_TABLE (fallback)
> - Wind: use Didion lag-time method

---

## WORK GROUP 5: Guide Tab

### What this does
In-app instructions for truing both the app AND a Kestrel 5700 ballistic weather meter. Three sub-sections.

### Prompt for Bolt

> Create a Guide tab with three sub-section buttons at top: "Truing", "Match Day", "Why".
>
> **Truing section** — unified instructions for calibrating both the TM22 app and a Kestrel 5700 in a single range session:
>
> Before the Range: create profiles, chrono 10 rounds, run tall target test, pick one chronograph.
>
> At the Range — show each distance as a step. For each step, show two color-coded lines:
> - Blue label "APP:" — what to enter in the app
> - Yellow label "KESTREL:" — what to adjust on the Kestrel
>
> Steps: Setup (record temp, DA, enter in app, confirm Kestrel has real MV) → Zero 50yd → 125yd (Kestrel: tune SHORT profile G7 BC starting at 0.078) → 150yd → 200yd (Kestrel: tune LONG profile G1 BC starting at 0.145, final lands ~0.50-0.55) → 225yd → 250yd.
>
> Show two side-by-side cards for Kestrel profiles:
> - SHORT (50-150yd): G7, BC ~0.078, trued at 125yd
> - LONG (150-300yd): G1, BC ~0.50-0.55, trued at 225yd
>
> Key rules: MV and scope height are measured facts. BC is the tuning knob. Never touch MV to make predictions fit. Cold barrel first for long distances.
>
> **Re-Truing section** — CRITICAL DISTINCTION:
> - App (blue box): Almost never needs re-truing for weather. TM22 is Mach-indexed — just update temp/DA in HUD. Only re-true for physical changes (new ammo lot, new scope mount, barrel wear).
> - Kestrel (yellow box): Needs re-truing when DA shifts >2000ft or temp changes >30°F. G1/G7 BC was trued at a specific Mach at a specific distance — conditions shift the Mach, making the BC wrong. This is TM22's core advantage.
>
> **Match Day section:**
> - Pre-match checklist: Kestrel on, read temp/DA, enter in app, confirm zero
> - Trust hierarchy: Below 150yd → Kestrel SHORT. 150-275yd → trust app (±0.03-0.14 MIL). Above 275yd → average app and Kestrel LONG. If they disagree >0.3 MIL → re-check zero and DA.
> - DA update rules: below 150yd don't bother, 150-225yd update if >500ft shift, 225-300yd update if >300ft shift
> - Quick reference table: distances, rounds per distance, purpose
>
> **Why section:**
> - Why two Kestrel profiles: G7 has flatter error shape at high Mach (short range), G1 becomes less wrong in deep subsonic (long range)
> - Why the app is better: TM22 indexes by Mach not distance. Same distance = different Mach on different days. Peak correction shifts 55 yards across realistic DA range. Kestrel applies correction at wrong place when conditions change.
> - What TM22 is: 59 Cd points from Doppler lab data, 85% Mach coverage, 77% solver noise reduction vs G1
> - Cross-condition validation table: Session 2→3 predictions, show ±0.03-0.14 MIL mid-range accuracy

---

## WORK GROUP 6: BLE Service + Tab

### What this does
Bluetooth Low Energy connections to four devices: TM22 DOPE display card, Calypso wind meter, Kestrel 5700, and Sig Kilo 3K rangefinder. Uses the @capacitor-community/bluetooth-le plugin.

### Prompt for Bolt

> Install @capacitor-community/bluetooth-le and create a BLE service layer.
>
> **BLE Service object** with methods: init(), scan(nameFilter, timeout), connect(deviceId, type), disconnect(deviceId), read(), write(), startNotify(). The service checks if Capacitor native platform is available — if not (browser), it gracefully shows "BLE not available."
>
> **Four device profiles with UUIDs:**
>
> TM22 DOPE Card (custom ESP32 device):
> - Service: 4fafc201-1fb5-459e-8fcc-c5c9c331914b
> - Write DOPE: beb5483e-36e1-4688-b7f5-ea07361b26a8
> - Read weather: beb5483f-36e1-4688-b7f5-ea07361b26a8
> - Read IMU: beb54840-36e1-4688-b7f5-ea07361b26a8
> - Shot notify: beb54841-36e1-4688-b7f5-ea07361b26a8
> - Name filter: "TM22"
>
> Calypso Wind Meter (standard BLE Environmental Sensing 0x181A):
> - Temp: 0x2A6E (int16 / 100 = °C)
> - Pressure: 0x2A6D (uint32 / 10 = Pa)
> - Humidity: 0x2A6F (uint16 / 100 = %)
> - Wind Speed: 0x2A72 (uint16 / 100 = m/s)
> - Wind Direction: 0x2A73 (uint16 / 100 = degrees)
> - Name filter: "CALYPSO"
>
> Kestrel 5700 (proprietary):
> - Service base: ffffffff-c446-4c4f-8c64-000000000000
> - Name filter: "Kestrel"
> - Individual characteristic UUIDs TBD (placeholder — needs nRF Connect mapping)
>
> Sig Kilo 3K (BDX protocol):
> - Service: 0000fff0-0000-1000-8000-00805f9b34fb
> - TX (write): 0000fff1-...
> - RX (notify): 0000fff2-...
> - Name filter: "KILO"
>
> **DOPE Packet Builder** — builds binary packets for the ESP32 display:
> - [stageName 16B][totalRounds 1B][numEngagements 1B][windMph 1B][tempF 1B]
> - Per engagement: [label 8B][posLabel 8B][distYd 2B LE][shots 1B][elevCentimil 2B LE signed][windCentimil 2B LE signed][confidence 1B][windDir 1B][targetSizeIn 1B]
>
> **BLE Tab UI:**
> - Four device rows: icon, name, connect/disconnect button
> - Status indicator showing connection state
> - When Calypso connects: subscribe to all 5 characteristics, parse data, pass weather updates to HUD via callback
> - Live weather display card showing temp, pressure, humidity, wind speed, wind direction when connected
>
> **HUD integration:** When BLE weather comes in, auto-fill temp and DA fields. Show green dot on ENVIRONMENT header when live data is streaming. Compute DA from barometric pressure and temperature.

---

## WORK GROUP 7: Capacitor Build + APK

### What this does
Wraps the web app into an Android APK using Capacitor.

### Prompt for Bolt

> Add Capacitor to this project for Android deployment.
>
> Install: @capacitor/core, @capacitor/cli, @capacitor/android, @capacitor/status-bar, @capacitor-community/bluetooth-le
>
> Configure capacitor.config.json:
> - appId: com.tm22.ballistics
> - appName: TM22 Logger
> - webDir: dist (Vite output)
> - backgroundColor: #0f172a
> - androidScheme: https
>
> Set up the build pipeline:
> - `npm run build` → Vite builds to dist/
> - `npx cap sync android` → copies dist/ to Android project
> - `npx cap run android` → deploys to connected phone
>
> Add safe-area CSS for Android notch/navigation:
> ```css
> body {
>   padding-top: env(safe-area-inset-top);
>   padding-bottom: env(safe-area-inset-bottom);
> }
> ```
>
> The BLE plugin auto-registers its Android permissions during cap sync — no manual AndroidManifest.xml edits needed.
>
> Test: build and install on Android phone. Verify all tabs render, localStorage persists across app restarts, and BLE tab shows "BLE not available" in browser but activates on the real device.

---

## WORK GROUP 8 (FUTURE): Stage Builder + Wind Bracket

### Not for initial build — describe for context only

> Stage builder: define NRL22 stages with engagement order, target distances, shots per target, position labels (prone, barricade, kneeling). Stage sequence displays one target at a time in large text with auto-advance on shot detection.
>
> Wind bracket probability: given two wind estimates with frequency weighting, compute the hold that maximizes first-round hit probability. Uses convolution of Gaussian system dispersion with uniform wind uncertainty. Numerical integration across target geometry.

---

## Build Order

1. **WG1** — Engine (must be mathematically correct before anything else)
2. **WG2** — Profiles + persistence (need data to feed other tabs)
3. **WG4** — HUD (immediate visual payoff, tests the engine)
4. **WG3** — Truing (calibration against real data)
5. **WG5** — Guide (reference material, no dependencies)
6. **WG6** — BLE (needs Capacitor, test on real device)
7. **WG7** — Capacitor build (wrap it all up)
8. **WG8** — Future features

## Verification

After each work group, test:
- WG1: Console test — 200yd should produce ~7.3 mils with TM22, ~6.9 with G1
- WG2: Create rifle + ammo, refresh page, data persists
- WG3: Enter dial-up at 200yd of 7.30 MIL → solved TM22 BC should be ~0.12-0.13
- WG4: Select rifle/ammo, enter 200yd → see elevation. Change temp/DA → elevation changes.
- WG5: All three sections render, no broken layouts
- WG6: BLE tab shows "not available" in browser. On Android: scan finds devices.
- WG7: APK installs, all tabs work, data persists across app kill

## Critical Rules for Bolt

1. **The TM22_TABLE and G1_TABLE must be copied exactly.** These are measured data. Do not round, do not reformat, do not "clean up."
2. **Scope height is in inches, measured center-bore to center-scope.** CZD = 2.6". This is NOT 1.5".
3. **MV is never a tuning knob.** The user measures MV with a chronograph. It's a fact. BC is what the solver adjusts.
4. **The solver uses TM22_TABLE when truing data exists, G1_TABLE as fallback when no truing data.**
5. **All units display in the rifle's selected turretUnit (MIL or MOA).** The engine always works in mils internally and converts for display.
6. **localStorage saves on EVERY state change.** No "save" button. Automatic.
7. **No confirm() dialogs.** They're blocked in WebView. Use inline double-tap confirmation instead.
8. **input type="number" is unreliable in WebView.** Use type="text" with inputMode="decimal" for numeric inputs.
