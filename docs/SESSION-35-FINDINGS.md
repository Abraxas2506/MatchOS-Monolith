# Session 35 Findings — Wind Log, Dev Tab, Field Data S4, UX Fixes

**Date:** February 17–18, 2026
**Context:** S4 field session (10 distances, DA 172, 64°F, est MV 1060), UX improvements, wind log system, dev notes tab, comprehensive feature planning
**Applies to:** BOLT-BUILD-GUIDE.md (all WGs), HANDOFF-22LR-BALLISTIC-CALC.md, SESSION-34-FINDINGS.md

---

## FIELD DATA — SESSION 4

**Date:** Feb 17, 2026 (evening)
**Conditions:** 64°F, DA 172, no chrono until after (Athlon read 1070, estimated MV used: 1060)
**Rifle:** CZD, 50yd zero, scope height 2.6"
**Ammo:** SK Rifle Match, lot 844

### Raw Dial-Ups

| Dist | Dial (MIL) | Pred | Error |
|------|-----------|------|-------|
| 75   | 0.70      | 0.74 | +0.04 |
| 100  | 1.90      | 1.78 | -0.12 |
| 125  | 3.00      | 2.97 | -0.03 |
| 150  | 4.10      | 4.26 | +0.16 |
| 200  | 7.10      | 7.08 | -0.02 |
| 225  | 8.60      | 8.59 | -0.01 |
| 250  | 10.20     | 10.18| -0.02 |
| 275  | 11.80     | 11.83| +0.03 |
| 300  | 13.50     | 13.55| +0.05 |
| 302  | 13.80     | 13.69| -0.11 |

### S4 Analysis

- S4 200–300yd: avg ±0.04 MIL — best session yet
- Estimated MV (1060) matched reality better than Athlon chrono (1070)
- Confirms Athlon reads ~10fps hot vs Caldwell-equivalent baseline
- Temp comp formula: 1056 + (64-59)*0.8 = 1060 — accidentally perfect

### Chrono Decision

User chrono'd AFTER shooting: Athlon read 1070. Data was entered with estimated 1060 (Chrono MV left blank). 

**Decision: Leave as-is.** Points stored with mv=1060. Predictions matched reality at 1060. Changing to 1070 would lower BC to compensate and end up in the same place. The MV/BC pair is internally consistent.

### Cumulative Solver State After S4

- **Single TM22 BC: 0.0535**
- **28 points across 4 sessions**
- **avg ±0.11 MIL, max ±0.30 MIL**
- DA range: -328 to 1007 ft (1335ft spread)
- S4 contributed 10 high-quality points including first 300yd data
- Max error (±0.30) is old S1 150yd point (4.05 actual, no chrono, DA -328)

---

## NEW FEATURES ADDED

### 1. Data Validation Gate

**Single point entry:** When user taps Add Point, app checks:
- BC clamped high/low/negative → warned
- Factor >2.0 or <0.15 → warned  
- Factor >1.0 → warned (unusual)
- >0.5 MIL off current single BC prediction → warned
- >0.3 MIL off → softer warning
- Dial-up doesn't match distance (too high/low) → warned

If any warnings fire, yellow box appears with specific messages. User must tap "Save Anyway" or "Cancel". Prevents garbage from entering the dataset.

**Collection mode:** Auto-rejects hard garbage (clamped BCs, factor >2.0). Reports what was skipped. Points >0.5 MIL off prediction flagged as low-conf but kept.

### 2. Number Keyboards

All numeric input fields now use `inputMode="decimal"` or `inputMode="numeric"` with `type="text"`. Phone pulls up number pad instead of full keyboard on:
- Distance, dial-up, shots, group size (truing)
- Chrono MV, temp, DA (truing + HUD)
- Tall target fields (test dist, dialed, measured)
- Wind speeds (HUD)
- Range (HUD)
- Collection mode dial-up, shots, group fields

InputRow component updated: always renders `type="text"` with explicit `inputMode` prop.

### 3. Truing Tab Persistence

Rifle selection, ammo selection, temp, DA, chrono MV, and dial unit now persist in localStorage under `tm22_truing_session` key. Survives tab switches and app restarts.

Auto-selects if only one rifle/ammo exists. Validates that persisted IDs still exist in current data (handles deleted profiles).

### 4. Scope Tracking Tool (Expanded)

- Opens automatically if tracking hasn't been tested (red "NOT TESTED" badge)
- Inline instructions explaining tall target test with worked example
- Shows expected displacement as user fills in dialed amount
- Color-coded result: green ±2% (excellent), yellow ±5% (acceptable), red worse (check mount)
- "Save to Rifle Profile" button (clearer than just "Save")
- Factor stored on rifle object, applied to all dial-up corrections

### 5. MV Labels Clarified

**Truing tab:**
- Chrono MV field gets green border when filled
- Green text: "✓ CHRONO: Using 1068 fps as-is — no temp adjustment"
- Yellow text when blank: "⚠ NO CHRONO: Using profile 1056 fps + temp adj (4.0 fps). Enter chrono'd MV for best accuracy."

**HUD tab:**
- Same green/yellow pattern with colored status bar
- Estimated MV shows breakdown: "1056 base +4.0 temp"
- Helper text under each field: "Measured today — used directly" vs "Leave blank if no chrono"

### 6. Legends

**Single TM22 BC panel:**
- Error color thresholds: green <0.1, yellow 0.1–0.2, red >0.2 MIL
- +/− meaning: over-predict (dial less) vs under-predict (dial more)
- BC and Sessions definitions

**TM22 Factor Curve panel:**
- Mach, Factor, Eff BC, Conf column definitions
- Normal factor range ×0.3–0.5
- Confidence symbols: green ● high, yellow ● moderate, red ● low
- ⚠ low-confidence zone explanation
- Count notation (3× = 3 data points)

### 7. Wind Log System

New collapsible section in HUD below Range Table. Persists in localStorage.

**Entry workflow:**
1. Set range, enter wind speed/direction in HUD
2. App shows predicted drift in blue box
3. Dial elevation only, shoot, read horizontal splash offset
4. Enter actual drift + direction in Wind Log
5. App stores: distance, wind call, predicted drift, actual drift, delta, TOF, conditions, note

**Calibration panel:** After 3+ observations with wind predictions, shows:
- Correction factor (actual/predicted average)
- Interpretation: "model under-predicts" / "model over-predicts" / "model is close"
- Avg and max error

**Observation grid:** Reverse chronological, shows Pred/Actual/Delta with color coding. Individual delete buttons.

### 8. Dev Tab

New 🔧 Dev tab at end of tab bar. Three sections:

**YOUR NOTES** — open notes displayed at top. Tagged as 🐛 Bug, 💡 Feature, ⚡ Improve, 📝 Note. Tap ○ to resolve, ✕ to delete. Timestamped, persisted.

**DEVELOPMENT ROADMAP** — three tiers:
- ▶ NEXT UP: spin drift, horizontal logging, cant diagnostic, wind adaptation
- ✓ COMPLETED: 16 features
- ◻ FUTURE: 17 planned features

**BUILD STATS** — current version, engine, accuracy, data summary

**ADD NOTE** — tag picker + textarea at bottom

**RESOLVED** — completed notes collapsed at bottom

### 9. Spin Drift + Cant Warning

**Miller spin drift formula** added to engine:
- `Sd = 1.25 * (SG + 1.2) * TOF^1.83`
- SG (gyroscopic stability) from twist rate, bullet weight, diameter (0.224"), length (0.43")
- CZD 1:16 RH twist, 40gr SK → ~0.02 MIL R at 200yd, ~0.06 MIL R at 302yd

**HUD display:**
- Spin drift shown below solver info: "Spin drift: 0.06 MIL R · SG: 1.43 · +0.65""
- Cant warning when spin drift significant: "At 13.8 MIL elev, 1° cant adds 0.24 MIL horizontal shift"

**Range table:** New "Spin" column shows drift at every distance with direction (e.g. "0.04R")

**Rifle profile:** Twist direction picker (Right/Left) added. Determines spin drift direction.

**Cant effect function:** `calcCantEffect(elevationMils, cantAngleDeg)` — computes horizontal shift and vertical loss from scope cant at any elevation. Used in diagnostic display.

### 10. Guide Tab Updated

- Truing section rewritten for single BC workflow
- Distance priority table (skip ≤125yd for truing, focus 175yd+)
- Chrono emphasis throughout
- Key rules updated: chrono bypass explained, data validation mentioned, more sessions > better math
- Match day: chrono'd vs estimated MV clearly differentiated
- Quick ref table updated with better distance priorities
- TM22 description updated with single BC solver info and current accuracy

---

## ARCHITECTURE DECISIONS

### Chrono Strategy

**Always chrono. Always enter the number. Both tabs.**

Estimated MV (profile + temp comp) is a fallback only. For truing sessions, it's not good enough — the whole point is accurate data.

Exception: S4 data was entered without chrono because the estimated MV (1060) happened to match reality. The Athlon's 1070 reading was discarded as ~10fps hot relative to the Caldwell baseline. The solver stores each point with its own MV, so this is safe.

### Short-Range (<125yd) Not Worth Fixing

BC model error at 75yd (0.07 MIL) is smaller than measurement noise (0.2 MIL spread). No algorithm — BC, interpolation, spline — can predict short range more precisely than turret resolution and group dispersion allow. This is a data collection problem (camera group analysis), not a math problem.

### Wind: The Real Competition Differentiator

Elevation with BC 0.0535 and avg ±0.11 is solved well enough. Wind is 80% of misses at distance. The wind log captures data to calibrate the Didion lag-time model. The correction factor absorbs systematic bias from terrain, reading habits, and all unmeasurable variables.

---

## PLANNED FEATURE ARCHITECTURE

### Spin Drift + Horizontal Calibration (NEXT)

**Layer 1 — Spin drift physics:**
- Miller spin drift formula: drift = f(twist_rate, bullet_length, velocity, TOF)
- CZD 1:16 twist, 40gr SK → ~0.7" at 300yd
- Displayed in HUD alongside elevation

**Layer 2 — Horizontal offset logging in truing:**
- At each distance, level rifle, shoot, log horizontal POI in MIL
- Stored alongside elevation truing points
- Builds horizontal offset curve by distance

**Layer 3 — Diagnostic alert:**
- Compare logged horizontal offset vs calculated spin drift
- If actual exceeds spin drift by threshold: "Horizontal offset at 250yd is 0.5 MIL beyond spin drift — check cant, windage zero, or mount alignment"
- Fires automatically with 3+ horizontal observations

**Layer 4 — Clean wind baseline:**
- Once spin drift and cant are accounted for, remaining horizontal misses in wind are pure wind model error + shooter calling error
- Wind log correction factor now means something real

### Wind Model Adaptation (NEXT)

Variables identified by user:
1. Wind at target ≠ wind at shooter position
2. Shot angle (azimuth + elevation) changes wind effect
3. Max ordinate changes with elevation angle (tower shots)
4. Shooter wind-calling skill improves over time → model should adapt
5. Bullet drift from muzzle wind vs terminal wind differs trigonometrically
6. Terrain effects (berms, tree lines, buildings create eddies)
7. Vertical wind component (updrafts/downdrafts on hillsides)
8. Air density variation across flight path (mirage over asphalt)
9. Time lag — wind changes during 0.3–0.5s flight time
10. Spin drift stacks with wind

**Approach:** The wind log captures actual results so the app learns the shooter's systematic bias. The correction factor absorbs terrain, reading habits, spin drift, and everything else into one number — same principle as BC absorbing all drag physics into one number.

### Camera Group Analysis (FUTURE)

- Target detection using known geometry (NRL22 standard targets)
- Scale calibration: target width in pixels → pixels per inch → pixels per MIL
- Shot hole detection on paper/cardboard
- Centroid calculation = POI
- POA reference from crosshair placement or target center
- Output: exact elevation + windage offset in MIL to 0.01 precision, group size
- Replaces manual dial-up entry (±0.1 MIL turret resolution → ±0.01 MIL camera resolution)
- Feeds both truing solver (elevation) and wind calibration (horizontal splash)

---

## BOLT SYNC STATUS

Bolt is currently at pre-Session-34 state. When credits reset, sync using:

1. **BOLT-BUILD-GUIDE.md** — 8 work groups with exact prompts
2. **SESSION-34-FINDINGS.md** — bugs, single BC solver, air density fix
3. **SESSION-35-FINDINGS.md** (this document) — S4 data, wind log, dev tab, all UX fixes
4. **HANDOFF-22LR-BALLISTIC-CALC.md** — complete system state (needs update to Session 35)

Feed Bolt the build guide WGs in order, applying Session 34+35 corrections. That gets Bolt to parity with the current APK.

New features (spin drift, horizontal logging, wind adaptation) should be built here in Claude R&D first, validated in the field, then ported to Bolt.

---

## FILES

- **APK:** `C:\projects\tm22-capacitor\tm22-app\android\app\build\outputs\apk\debug\app-debug.apk`
- **Source zip:** `tm22-v3-dev.zip` (includes wind log + dev tab)
- **This doc:** `SESSION-35-FINDINGS.md`
- **Previous:** `SESSION-34-FINDINGS.md` (still applies — bugs, single BC)
- **Bolt guide:** `BOLT-BUILD-GUIDE.md` (needs Session 35 addendum)
- **Handoff:** `HANDOFF-22LR-BALLISTIC-CALC.md` (needs Session 35 update)
