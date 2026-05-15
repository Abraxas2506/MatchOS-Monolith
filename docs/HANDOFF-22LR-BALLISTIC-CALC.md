# .22 LR Ballistic Calculator & Logger — Handoff Document

**Date:** February 17, 2026  
**Status:** Active development — sessions spanning Feb 5–17, 2026  
**Current build:** V3 with single TM22 BC engine + BLE + Guide tab + Capacitor  
**Branding:** "TM22 Ballistic Logger"  
**Match deadline:** ~Feb 21, 2026  
**Context note:** Chat context limits hit once:  
1. **Chat 5 (Feb 11):** Hit limit mid-extraction of Range Buddy APK (group analysis computation classes).  
Chat 6 (Feb 12-13): Compacted mid-session. Merged two transcripts. Added Guide tab, fixed re-true rules, built Capacitor Android project with BLE service layer.  
Chat 7 (Feb 16): Bolt Build Guide created (8 work groups with exact prompts). Complete handoff rebuilt.
Chat 8 (Feb 17): Session 3 field data analyzed. MV double-counting bug found/fixed. Single TM22 BC solver replaces stepped BC. Session 34 findings document created with Bolt corrections.

---

## CRITICAL CONTEXT

### Three App Versions Exist
- **V1 (~11,400 lines):** Original monolith, full feature set. Ported to Capacitor at `C:\projects\22_Ballistic_Logger`. APK built. Archived as reference.
- **V2 (~920 lines):** Simplified rebuild, three tabs (HUD, Truing, Profiles). Stepped BC engine only. Superseded by V3.
- **V3 (current):** Single TM22 BC engine (replaces stepped BC). Five tabs: HUD, Truing, Profiles, BLE, Guide. localStorage persistence. BLE service layer (DOPE card, Calypso, Kestrel, Sig Kilo). Capacitor Android wrapper with APK build. Bolt Build Guide written (8 work groups). Session 34 findings document with Bolt corrections.

### Architecture: Capacitor (NOT React Native)
- React Native is dead (Expo SDK 52 broken, Node version issues)
- Capacitor = Vite + React + Tailwind wrapped in Android native shell
- Same architecture as user's working Wind Bracket Calculator app

### "Bolt Owns the Code" Strategy
- Bolt can store 12K lines but can't intelligently modify code it didn't write
- Rebuild plan: 7 task groups, Bolt writes every line from descriptive prompts
- V2 preview validates math/UX in Claude first, then Bolt rebuilds from prompts
- User can use Opus 4.6 in Bolt

---

## SCOPE HEIGHT (CRITICAL)

**CZD scope height is 2.6", measured with digital calipers. ALWAYS 2.6".**
**CZT scope height is ~2.0" (stored in Kestrel, needs verification).**

Early sessions incorrectly used 1.5" for CZD. All Session 1 solved BCs computed with 1.5" are INVALID. Session 2 and forward use the correct 2.6". Any old truing data must be cleared and re-entered.

---

## ENGINE ARCHITECTURE

### Current Engine: Single TM22 BC (V3 App, Feb 17+)
- **Core:** AB SK Match CDM Doppler data, polynomial-smoothed — 59 Cd points spanning Mach 0.625-0.875 (85% coverage)
- **Prediction engine:** `solveSingleTM22Bc(points)` — one BC value fits ALL truing points across ALL sessions simultaneously
- Each point evaluated under its OWN recorded conditions (temp, DA, MV), weighted by confidence
- Golden section search minimizing total squared error, BC range [0.01, 0.15]
- **Validated:** avg ±0.13 MIL, max ±0.23 across 16 points, 3 sessions, 1335ft DA swing
- **Chrono'd MV bypass:** when user enters chrono'd MV, temp compensation is disabled (temp=59 passed to solver)
- **Air density:** ISA power law `Math.pow(1 - DA / 145442, 4.256)` — NOT exponential
- **Temp comp:** `adjustedMV = MV + (temp - 59) × 0.8` — only applied when using profile baseline MV, not chrono'd MV
- Factor curve display and per-point BCs retained for diagnostics (Truing tab), but NOT used for prediction

### Previous Engine: Stepped BC (V3 pre-Feb 17, V2)
- Each truing point independently solves its own flat BC via binary search (`solveBcFromDialUp`)
- `lookupSteppedBc(dist, points)` interpolates between solved BCs by distance
- **Mathematically guaranteed** to reproduce exact dial-ups at truing distances
- **PROBLEM:** Breaks across conditions — Mach at 225yd changes with DA/temp, so distance-indexed BCs apply at wrong Mach
- Cross-condition accuracy: ±0.18 MIL (with chrono fix), ±0.35 MIL (with MV double-count bug)
- **Retained** only for Truing tab factor curve display, NOT used for HUD predictions

### TM22 Informational Display
- Factor = solved_BC / base_BC at each point's terminal Mach number
- `buildTM22Display()` builds Mach-indexed table for UI visualization
- Shows the Cd correction curve shape — useful for understanding bullet behavior

### Why TM22 Mach-Variable Integration FAILED
Multiple attempts were made (iterative, sequential inside-out, wider bounds) — all failed:
- Each truing point's BC was solved independently using flat-BC solver
- When applied as Mach-variable factors during integration, the factors compound
- Close-range inflated factors suppress early-flight drag, cascading into wrong predictions at distance
- Sequential inside-out solver also failed — oscillation between ceiling and floor clamps
- Root cause: converting flat-BC solves into point Mach-corrections is an ill-conditioned inverse problem
- **Proper solution requires simultaneous nonlinear optimization (Levenberg-Marquardt)**
- The stepped BC approach was reverted to because it actually works — every truing distance reproduces exactly

### Solver Iteration History (for context on what was tried)
1. Independent flat-BC solve per point → factors stored → TM22 Mach-variable integration → **HUD showed 13.06 vs 13.7 target** (factors not self-consistent)
2. Split raw/display tables to avoid merge interference → **13.19** (stale Mach indexing)
3. Mach update per iteration → **13.53** (factor bounds 0.5-2.0 too tight)
4. Widened bounds to 0.3-3.0 → **13.23** (still clamping)
5. Widened to 0.1-5.0 → **solver oscillating** (100yd point ×2.78 poisoning curve)
6. Excluded <2.5× zero, clamped 0.5-2.0 → **20.72** (only 2 points, stale factors)
7. Sequential inside-out solver → **15.78** (factors diverging: ×5.0 ceiling, ×0.2 floor)
8. **Reverted to stepped BC** → **13.70 exactly** ✓

---

## DRAG MODEL HISTORY & DECISIONS

### Timeline
1. **Feb 5-6 (V1):** G7 engine built, then G1 added. User selects per profile.
2. **Feb 7 early:** Researched manufacturers. All publish G1. Locked to G1 only.
3. **Feb 7 mid:** G1 solver producing inflated BCs. Investigated RA4.
4. **Feb 7-8:** RA4 research — **identical to G1 below Mach 1.23** (1395fps). Dead end.
5. **Feb 7-8:** TM22 concept invented — custom drag model from field data.
6. **Feb 7-8:** TM22 Mach-variable integration attempted → **FAILED** (see above).
7. **Feb 8-9:** TM22 kept as **informational display only**. Stepped BC is the engine.
8. **Feb 9-10:** Cross-condition validation (Session 3). Mid-range ±0.06-0.12 MIL. Short-range diverging.
9. **Feb 10:** TM22 reference curve with G7/G1 scaffolding built → **77% solver noise reduction**.
10. **Feb 10:** AB CDM extraction (5 ammos including Norma), two TM22 families identified, comparison tool built.
11. **Feb 10:** TM22 v2 — AB SK Match CDM polynomial-smoothed replaces field data + all scaffolding. 59 lab points, 85% Mach coverage.

### Current: TM22 v2 Reference Curve Engine (V3)
- Engine uses TM22 v2 Cd(Mach) reference table — AB SK Match CDM, polynomial-smoothed, 59 points
- 85% Mach coverage from Doppler lab data, ~15% remaining scaffolding at extreme edges
- Confidence zones displayed: green (lab-validated), yellow (scaffolding)

### RA4 Research Summary
- Multiple sources claim "identical to G1 below 1400fps"
- Airgun Nation data shows Cd difference at Mach 0.8 (RA4: 0.2330, G1: 0.2546) — but these may use different normalization (different reference projectile shapes)
- RA4 reference projectile is "40 grain .22 Rimfire solid projectile"
- JBM Ballistics has RA4 table but couldn't fetch it (network blocked)
- **Bottom line:** For standard velocity 22LR (always below Mach 1.0), RA4 offers no meaningful improvement over G1. The "identical below 1400fps" claim from RSI may mean curve shape, not absolute Cd values.

### Published BCs (Reference)
| Manufacturer | Product | G1 BC | Notes |
|-------------|---------|-------|-------|
| Lapua | All 22LR | 0.172 | Umbrella value |
| SK | All | 0.172 | Made by Lapua/Nammo |
| SK Standard+ | Sniper's Hide | 0.132 | User-reported |
| Eley | Match/Club | 0.140-0.150 G1, 0.112 RA4 | Only one publishing RA4 |
| RWS | R50/R100 | 0.105-0.109 | |
| Norma | Match-22 | 0.132 | Made by RWS |

### BC Value Confusion (RESOLVED)
- User enters BC 0.135 G1 in profile
- SK/Lapua publishes 0.172 G1 (some sources); Sniper's Hide reports 0.132
- Both values exist — the "correct" BC depends on measurement distance/method because G1's curve shape is wrong for 22LR
- G7 0.080 (user's AB tune) converts to G1 0.173-0.200 — matches published 0.172
- Solved G1 0.269 at 147yd converts to G7 0.105-0.120 — does NOT match user's 0.08
- **The G7 0.08 and published G1 0.172 describe the same bullet at similar velocities**
- The solver's inflated G1 values (0.26-0.56) are because G1 overpredicts drag for this bullet

---

## APPLIED BALLISTICS DSF (Research)

### How AB's Drop Scale Factor Works
- DSF is a straight percentage multiplier on predicted drop at a specific Mach number
- Up to 6 DSF calibration points through transonic/subsonic
- AB workflow: true MV in supersonic range, true BC via DSF in transonic
- Doesn't change MV or BC — just scales the output

### Why DSF Fails for 22LR
- AB designed for centerfire starting supersonic, going transonic at distance
- 22LR starts at Mach 0.94 — already in the problem zone
- "True MV in supersonic, then DSF in transonic" doesn't map to 22LR
- Chris Simmons' workaround: zero 50, true velocity 200, true BC 300 — manual curve fitting

---

## USER'S PHILOSOPHY ON VARIABLES

This is critical context for any future work:
- **MV is a FACT** — measured with chrono, consistent instrument error, within 5fps of other chronos
- **Scope height is a FACT** — measured with digital calipers (2.6")
- **BC is NOT a fact** — varies with conditions, drag model is wrong
- **DA is NOT a fact** — varies with conditions
- User **refuses to fudge known-good variables** (MV, scope height) to compensate for unknown variables (BC, DA)
- This is why Chris Simmons' "adjust MV until it fits" method works but isn't scientific
- User wants to **solve the actual unknowns**, not hack the knowns

---

## SHORT-RANGE PROBLEM (50-100yd)

### The Issue
- User needs two profiles in AB: one for 50-100yd (G7 ~0.08), one for 100-300yd
- At 50-100yd from 50yd zero, total drop is only ~6 inches. BC barely matters.
- What dominates: MV, scope height, gravity
- "Truing BC at 100yd" in AB is really compensating for MV/scope-height error using BC as a fudge knob

### 100yd Too Close to 50yd Zero for BC Solving
- Total sight-line correction at 100yd is only ~6 inches
- Changing BC from 0.15 to 0.30 only changes predicted drop by ~0.15 mils (~0.5 inches)
- A 0.1 mil measurement error swings solved BC by ±0.15
- Physics floor: even BC=0.3 (vacuum trajectory) produces ~2.06 mils at 100yd
- Minimum useful truing distance: **2.5× zero** (125yd for 50yd zero)

### Solution in V3 (TM22 Reference Curve)
- TM22 reference curve reduces solver noise by 77% — short-range BCs become physically reasonable
- At 75yd: TM22 solves BC ~0.098 (factor ×0.729) instead of G1's 0.425 (factor ×3.149)
- Atmospheric scaling on a reasonable BC produces much smaller cross-condition error
- G7 scaffolding above Mach 0.84 fills the short-range gap with a shape that matches user's competition-validated G7 0.078 tune
- **Cross-condition accuracy still weaker at short range** — Session 3 showed +0.18-0.22 MIL at 75-125yd vs ±0.03-0.14 at mid-range
- **Empirical DA correction rate** being built from multi-session 75yd control point data (-0.48 MIL per 1000ft DA from 2 sessions)

---

## FIELD DATA

### User's Setup
- **Rifle 1 (CZD):** Scope height **2.6"**, barrel 20", twist 1:16 RH, 50yd zero
- **Rifle 2 (CZT):** Wife's rifle, scope height **~2.0"** (verify from Kestrel), barrel 20", twist 1:16 RH, 50yd zero
- **Ammo:** SK Rifle Match, lot 844, MV 1056 (Caldwell chrono), ES 28, SD 8
- **Location:** Savannah, Georgia
- **Base BC entered:** 0.135 G1

### Chronograph Baseline (CRITICAL)
- **Caldwell** is the baseline chronograph — Sessions 1 and 2 used Caldwell (1056fps)
- **Athlon** reads ~9fps higher (1065fps) — measured same-day, same-ammo offset in Session 2
- **Session 3** used Athlon only (1065fps) → corrected to Caldwell-equivalent 1056fps
- **At matches:** Athlon is used (Caldwell picks up other shooters). Both CZD and CZT will be fresh-trued on Athlon before the match.
- **Athlon is fine for SD/ES** — relative measurements where consistent offset doesn't matter
- **Rule:** Pick one chrono and use it exclusively for MV baseline. Currently standardized on Caldwell 1056 for stored data.

### Session 1 (Feb ~6, DA -591, Temp 53°F, MV 1056)
| Distance | Dial-Up (MIL) | Notes |
|----------|--------------|-------|
| 100 | 1.70 | Too close to zero, low confidence |
| 147 | 4.10 | |
| 192 | 6.55 | |
| 302 | 13.70 | Cold barrel — MORE TRUSTED for 302 |

**Note:** Originally solved with 1.5" scope height (wrong). All Session 1 BCs are invalid. Dial-ups are still good field data.

### Session 2 (Feb 8, DA -745, Temp 53°F, MV 1056 Caldwell)
| Distance | Dial-Up (MIL) | Solved BC (G1) | TM22 Factor | Confidence | Notes |
|----------|--------------|---------------|-------------|------------|-------|
| 75 | 0.80 | 0.4251 | ×3.149 | LOW (< 2.5× zero) | |
| 102 | 1.95 | 0.4660 | ×3.452 | LOW (< 2.5× zero) | |
| 125 | 3.05 | 0.5027 | ×3.724 | OK | |
| 150 | 4.35 | 0.5316 | ×3.938 | OK | |
| 175 | 5.70 | 0.5641 | ×4.179 | OK | **Peak factor** |
| 200 | 7.30 | 0.5549 | ×4.111 | OK | |
| 225 | 8.90 | 0.5384 | ×3.989 | OK | |
| 250 | 10.45 | 0.5349 | ×3.962 | OK | Slightly above trend |
| 275 | 12.45 | 0.4736 | ×3.508 | OK | |
| 302 | 14.70 | 0.4433 | ×3.284 | LOW | ±0.5 MIL, barrel heating suspected |

### Session 3 (Feb ~10, 3-hour session, Athlon chrono → corrected to Caldwell 1056)
**Conditions:** Start 58°F / DA -328 / MV 1060 (Athlon) → corrected 1056. End 53°F / DA -770 / MV 1055.

| Group | Distance | Dial-Up (MIL) | Notes |
|-------|----------|--------------|-------|
| 1 | Zero 50 | -0.05 | Confirmed |
| 1 | 302 | 14.0 | **Cold barrel — FIRST target of session** |
| 1 | 75 | 0.6 | |
| 1 | 150 | 4.05 | |
| 2 | 125 | 2.8 | |
| 2 | 175 | 5.6 | |
| 2 | 192 | 6.6 | |
| 2 | 225 | 8.7 | |
| 3 | 100 | 1.8 | Temps dropping |
| 3 | 250 | 10.1 | Dark, poor centering, high DA drift |
| 4 | 75 | 0.6 | **Reconfirmed — consistent** |
| 4 | Zero | 0.0 | **Reconfirmed** |

### Cross-Condition Validation Results (S2 → S3)
Engine predictions using Session 2 stepped BC table under Session 3 conditions:

| Distance | S3 Actual | S2 Prediction | Delta | Verdict |
|----------|-----------|---------------|-------|---------|
| 100 | 1.80 | 1.86 | +0.06 | ✓ Within resolution |
| 175 | 5.60 | 5.63 | +0.03 | ✓ Excellent |
| 192 | 6.60 | 6.69 | +0.09 | ✓ Within resolution |
| 225 | 8.70 | 8.84 | +0.14 | Close |
| 302 | 14.00 | 14.21 | +0.21 | Usable |
| 75 | 0.60 | 0.78 | +0.18 | Short-range divergence |
| 125 | 2.80 | 3.02 | +0.22 | Short-range divergence |
| 250 | 10.10 | 10.46 | +0.36 | Noise (dark, temp dropping) |

**Key findings:**
- **Mid-range (150-225yd): ±0.03-0.14 MIL** — model validated across conditions
- **Long range (302yd): +0.21 MIL** — usable, cold barrel value converging
- **Short range (75-125yd): +0.18-0.22 MIL** — stepped BC distorts atmospheric scaling (inflated BCs)
- **250yd: +0.36 explained** by dark conditions + poor centering + temp dropping. Not anomalous.
- **Consistent positive bias** across all points — predictions run slightly high

### 302yd Convergence (Three Sessions)
- Session 1: **13.7 MIL** (cold barrel, DA -591) — trusted
- Session 3: **14.0 MIL** (cold barrel, DA -328) — trusted, shot first
- Session 2: **14.7 MIL** (hot barrel, DA -745) — outlier, barrel heat confirmed
- S1 and S3 are both cold barrel, 0.3 MIL apart with 263ft DA difference — physically consistent
- App predicts 14.14 at S3 conditions — delta +0.14, very good

### TM22 Curve Shape (Session 2, 75-275yd)
- **Smooth and physically sensible bell shape**
- Rising leg: 75yd (×3.15) → peaks at 175yd (×4.18) at Mach 0.81
- Falling leg: 175yd (×4.18) → 275yd (×3.51)
- Tells real story: G1's transonic drag spike peaks at wrong Mach for this bullet
- **250yd:** Session 2 showed slight bump above trend. Session 3 at 250yd explained by dark/temp conditions. Bracket test at 237/262 still planned.

### Peak Shift Discovery (IMPORTANT)
- The peak correction factor ALWAYS occurs at Mach 0.812 regardless of conditions
- But the **distance** where Mach 0.812 occurs shifts with DA/temp:
  - Cold day (DA -1000): peak at ~163yd
  - Standard day (DA 0): peak at ~185yd
  - Hot/altitude (DA +3000): peak at ~218yd
- **55-yard swing** across realistic conditions
- This is why stepped BC (distance-indexed) breaks under condition changes — it applies peak correction at the wrong distance
- TM22 (Mach-indexed) doesn't have this problem — correction follows the bullet through the air
- **This is the core practical argument for TM22 over stepped BC**

### Short-Range Fix Strategy
- **Now (Option 1):** Empirical DA correction rate. 75yd data: 0.80 MIL at DA -745, 0.60 at DA -328 → -0.48 MIL per 1000ft DA. Needs 3+ sessions to confirm linearity.
- **Later (Option 3):** TM22 with mid-range extrapolation. Factor curve from 125-175yd climbs at known rate → project backward to 75-100yd Mach numbers. Highest risk but fixes actual physics.

### Next Session: Match Prep Truing (~Feb 14-18)
Both CZD and CZT will be fresh-trued on Athlon before the match:
1. For each rifle with lot 844: Zero at 50yd (5 rounds), chrono 10 rounds on Athlon, shoot 150yd + 200yd (10 rounds each)
2. If time: add 125yd and 250yd (50 total rounds per rifle)
3. This simulates a new user's first experience — proves TM22 works with any chronograph
4. CZD's Session 2 data kept as reference for comparison

### Match-Day Workflow (~Feb 21)
1. Enter temp, DA from Kestrel, MV from Athlon into app HUD
2. Confirm zero at 50yd
3. Select target distance → read elevation → dial → send
4. **DA update rule:** Glance at Kestrel between stages. Update if DA shifted >500ft (150-225yd) or >300ft (225-300yd). Below 150yd, don't bother.

### Two-Pass Experimental Design (Future, Post-Match)
- **Day 1 (low density, cold barrel):** 75, 150, 175, 200, 225, 302 — 6 anchor points, 60 rounds
- **Day 2 (high density, different day):** 155, 160, 165, 170, 180, 185, 190, 195, 205, 210, 215, 220 — 12 fill points, 120 rounds
- Low-density curve used as truth filter: any high-density point >±0.15 MIL from interpolated prediction gets flagged
- Different days is better than same day — eliminates barrel heat, tests atmospheric model

---

## WIND PHYSICS

### Model: Lag-Time (Didion)
- `drift = crosswind_fps × (TOF - distance / MV)`
- The lever arm effect dominates: early-flight wind push has full remaining distance to accumulate displacement
- Didion 70/30 weighting: wind near shooter matters ~2× more than wind at target
- User corrected Claude: "A small movement at the muzzle translated out to 300 yards becomes very large, the equal amount of push at 200 doesn't translate trigonometrically that much from 200 to 300"
- TM22/stepped BC improves wind calls automatically because TOF is more accurate

### Wind UI
- Target Bearing (°), Wind FROM (°), Wind Min (mph), Wind Max (mph)
- Clock/Degrees toggle with 3:15 format support
- Crossing angle computed from bearing difference
- Windage bracket: shows both MIN and MAX holds + midpoint favored hold

---

## V3 FEATURES (Current Build)

### Five-Tab Layout: HUD, Truing, Profiles, BLE, Guide

### HUD Tab
- Range with ▲ UP / ▼ DN elevation, +/- buttons using selected increment
- Environment: Today's MV (fps), Temp (°F), DA (ft) — humidity/pressure removed
- Range table: min/max/increment dropdowns, wind columns (W Min, W Max) when active
- Flight data: velocity, energy, TOF, Mach
- Wind section with clock/degrees toggle, crossing angle display
- Windage bracket: MIN and MAX holds + midpoint
- UNTRUED (yellow) / TRUED ×N (green) badge
- Units in rifle's selected unit (MIL/MOA)

### Truing Tab
- **Tall Target Test:** Dialed + unit picker, Measured + unit picker, saves tracking correction
- **Single Point Entry:** Distance, dial-up, shots, group size, chrono MV, conditions
- **Collection Mode:** Toggle between single-point and grid entry
  - 10 preset distances (75-300 by 25yd) with add/remove custom distances
  - Batch save with shared session ID and conditions
  - Session note field
  - Yellow highlight on distances below 2.5× zero
- **Truing Cards:** Editable (✎), deletable, show BC, TM22 factor, Mach, temp, DA, scope tracking
- **TM22 Drag Curve panel:** Informational display of Mach/Factor/Effective BC/Confidence
- Minimum distance warning: 2.5× zero (125yd for 50yd zero)
- Confidence weighting: weight = √shots / groupMOA
- Mach-band merging: points within 0.015 Mach merge via weighted average (display only)
- Low-confidence flagging for close-range points

### Profiles Tab
- **Rifle:** Name, zero (50yd), scope height, barrel length (20"), twist (1:16), click size, turret unit (MIL/MOA), scope tracking %
- **Active Profiles:** CZD (2.6" scope height), CZT (wife's, ~2.0" scope height TBV)
- **Ammo:** Profile name, manufacturer→product dropdown, lot #, MV, SD, ES, bullet weight, BC, group size + unit
- **Active Ammo:** SK Rifle Match Lot 844
- Drag model: TM22 reference curve (with G7/G1 scaffolding)

### BLE Tab
- Scan/connect to 4 device types: TM22 DOPE Card, Calypso Wind, Kestrel 5700, Sig Kilo 3K
- Each device row shows icon, name, connect/disconnect button, connection status
- Gracefully degrades in browser ("BLE not available")
- Calypso: subscribes to all 5 ESS characteristics, parses to °F/inHg/%/mph/degrees
- Weather auto-fills HUD temp and DA (computed from barometric pressure + temp)
- DOPE packet builder: binary format matching ESP32 firmware, sends stage data to display card

### Guide Tab
- Three sub-sections: Truing, Match Day, Why
- **Truing:** Unified session for app AND Kestrel — each distance shows blue (APP) and yellow (KESTREL) instructions side by side
- **Kestrel two-profile cards:** SHORT (G7 ~0.078, 125yd) and LONG (G1 ~0.50-0.55, 225yd)
- **Re-true rules:** App = almost never (Mach-indexed auto-adjusts). Kestrel = re-true for DA >2000ft or temp >30°F shift
- **Match Day:** DA update thresholds by distance band, trust hierarchy (Kestrel SHORT below 150, app 150-275, average above 275)
- **Why:** Peak shift discovery (55yd swing), G7/G1 error shape analysis, TM22 validation data

### Persistence
- localStorage key "tm22_data_v3"
- Saves rifles, ammo, truingData, active tab on every state change
- Loads on mount, survives app kill and phone restart

---

## BUGS FIXED (V2-V3, Feb 7-10)

| # | Bug | Fix |
|---|-----|-----|
| 1 | InputRow inside component — remounts on keystroke | Moved outside component |
| 2 | Add Point silent failure — type="number" flaky in artifact | Changed to inputMode="decimal" + error messages |
| 3 | DA input fighting user edits | Made direct input, no calculated value |
| 4 | Range input zero prefix stuck | Empty on blur → default 25yd |
| 5 | Tracking correction formula direction | `corrMils = dialMils / tc` (matching V1) |
| 6 | Edit truing data loss (CRITICAL) | Stale closure → functional setState `prev => ...` |
| 7 | Clock helpers type error | `.includes(":")` on number → `String()` coercion |
| 8 | BC solver clamping rejects valid entries | Changed to inline warning, saves anyway |
| 9 | confirm() dialogs blocked in iframe | Removed all — replaced with inline warnings, double-tap clear |
| 10 | Scope height 1.5" → 2.6" | Was always 2.6", entered wrong. All old solves invalid. |
| 11 | TM22 Mach-variable solver oscillation | Reverted to stepped BC. TM22 display-only. |
| 12 | 100yd excluded by 3× zero rule but 147yd also excluded | Relaxed to 2.5× zero (125yd minimum for 50yd zero) |

---

## TM22 — FUTURE DRAG MODEL

### Concept
- A reference Cd(Mach) table specific to 22LR round-nose heeled bullets
- Replaces G1/G7 as the drag model — correct curve shape for the actual bullet
- Each ammo gets a single "TM22 BC" that scales the curve
- One number, one truing point, full curve — like G7 for centerfire but accurate

### Endgame User Workflow (Once TM22 Published)
**Casual shooter:** Select ammo from database → app loads published TM22 BC → enter MV from chrono → done. No truing at all.
**Competitive shooter:** Same setup + shoot one 10-round group at 200yd → app solves lot-specific TM22 BC → done. 10 rounds total.
The two bricks of ammo being burned now build the foundation that saves every future user from ever doing it.

### Why It's Achievable
- 22LR bullets are the most geometrically consistent class: 40gr, 0.223", round nose, heeled base
- AB CDM data confirms: SK, Eley, Lapua cluster within 5fps at 200yd — **three manufacturers, one curve shape**
- RWS R50 is the outlier (26-31fps less drag) — needs separate curve family
- Cd(Mach) curve shape shared within each family, only BC scaling differs

### Two TM22 Families
- **TM22-S (Standard):** SK, Eley, Norma, Lapua — covers ~75% of competitive 22LR
- **TM22-R (RWS):** RWS R50/R100 — already available from AB CDM export (no range time needed)
- Minor caveat: Lapua Center-X curves cross SK Match at ~300yd. Small effect but may warrant sub-family.

### Extraction Methodology
1. Dense dial-up collection (75-300yd every 25yd) — **Session 2 done**
2. Solve BC at each distance → extract Cd correction factor at each Mach
3. Multi-session data at different DA separates bullet properties from atmospheric effects
4. 2×2 factorial (2 rifles × 2 ammos, same day) validates curve shape is shared
5. Fit smooth Cd(Mach) table with L-M optimizer
6. Validate: one TM22 BC + curve predicts trajectory at new conditions

### What's Needed
- **Phase 1 (DONE):** Single ammo, single rifle, dense collection → Session 2
- **Phase 2 (DONE):** Same setup, different DA day → Session 3 cross-condition validation PASSED (mid-range ±0.03-0.14 MIL)
- **Phase 3 (DONE — via AB):** AB CDM data for SK, Eley, Lapua, RWS confirms curve families
- **Phase 4 (next — match):** CZD + CZT trued fresh on Athlon, validated at competition
- **Phase 5:** L-M optimizer implementation → proper single-BC TM22 engine
- **Phase 6:** Publish table + validation data

### 2×2 Factorial Design
User proposed testing 2 rifles × 2 ammos on same day:
| | Ammo A | Ammo B |
|---|--------|--------|
| **Rifle 1** | Curve shape + BC_1A | Curve shape + BC_1B |
| **Rifle 2** | Curve shape + BC_2A | Curve shape + BC_2B |

If TM22 is valid, all four produce the same curve shape with different BC scaling. Same day eliminates atmosphere as variable. ~800 rounds, one afternoon.

### Current Curve Shape (Session 2)
- Bell-shaped: rises from Mach 0.86, peaks at Mach 0.81, descends to Mach 0.71
- Peak factor ×4.18 at 175yd — G1 is ~4× wrong at this velocity for this bullet
- Physically sensible: G1's transonic spike peaks at wrong Mach for round-nose 22LR
- 250yd anomaly (above trend) — user will bracket with 237/262 shots
- 8 of 9 points trace a smooth, physically sensible curve

### L-M Optimizer Path (Future)
- Cost function: sum of squared errors between predicted and actual dial-ups
- Jacobian: perturb each factor by 0.001, re-run all integrations, measure change
- JS library: ml-levenberg-marquardt (~5KB npm package)
- Should converge in 5-10 iterations, ~200ms total
- The real payoff is extrapolation — predicting 350-400yd from data that only goes to 302

---

## AB CUSTOM DRAG MODEL DATA (Feb 10 — MAJOR FINDING)

### Data Extracted
User exported AB CDM velocity curves from Kestrel for five ammos. Each export contains velocity at every 5 yards from 0-500yd. Cd extracted directly from deceleration between increments.

All tested at: 48°F, 29.65 inHg, BC 1.000 (raw drag model, no scaling), MV 1080fps.

### Velocity at 200yd (from 1080fps start):
| Ammo | Velocity at 200yd | Relative Drag |
|------|-------------------|---------------|
| RWS R50 | 833 fps | Lowest drag |
| Eley Match | 807 fps | Higher drag |
| SK Match | 806 fps | Higher drag |
| Norma Match | 805 fps | Higher drag |
| Lapua Center-X | 802 fps | Highest drag |

### TWO TM22 FAMILIES IDENTIFIED

**TM22-S (Standard):** SK, Eley, Norma, Lapua — four manufacturers converging within 5fps at 200yd. Close enough that one reference curve with different BCs per ammo works. Covers ~75% of competitive 22LR.

**TM22-R (RWS):** RWS R50/R100 — 26-31fps less drag than the cluster. Needs its own reference curve. AB's CDM velocity data IS the curve (no range time needed).

### Critical Finding: Lapua Crossover
Lapua Center-X and SK Match curves **cross** around 300yd:
- 50yd: Lapua 4fps slower than SK
- 200yd: Lapua 4fps slower
- 300yd: Lapua 2fps FASTER
- 350yd: Lapua 4fps FASTER

A single BC scaling factor cannot capture a crossover — this means Lapua and SK don't perfectly share curve shape. The crossover is small enough for competition use but matters for TM22 precision. Possible third sub-family within TM22-S.

### Why This Matters
- AB already did the lab work (Doppler radar) — user doesn't need to burn ammo for ammos they don't shoot
- AB's Cd vs Mach curve is dimensionless — lab conditions don't matter, curves are directly comparable to TM22 field data
- Validates/replaces scaffolding: if AB's curve agrees with TM22 empirical points in the overlap zone (Mach 0.735-0.838), their data can replace G7/G1 scaffolding
- **Endgame:** Every ammo with an AB CDM gets instant TM22-compatible profile. New shooter: select ammo, enter MV, done.

### Comparison Tool Built
- JSX artifact: TM22 vs AB CDM overlay chart with all 5 ammos
- Dropdown to compare TM22 empirical points against each CDM individually
- G1/G7 reference toggles
- Overlap zone zoom panel with percentage difference

### TM22 v2 Reference Curve (AB Lab Data Replacing Scaffolding)
- AB's SK Match CDM velocity export has integer-rounded fps at 5yd increments
- Integer rounding creates ±15% noise in raw Cd extraction (worse than field data)
- **Fix:** Degree-5 polynomial fit to velocity curve (max residual 0.52fps = just the rounding itself)
- Smooth analytical derivative → clean Cd extraction with zero oscillation
- **Result:** 59 lab-measured Cd points spanning Mach 0.625-0.875 (85% of range)
- Replaces: 6 noisy field points (13% coverage) + G7/G1 scaffolding
- Curve shape: gently rising from Cd 0.0647 (Mach 0.625) to 0.0743 (Mach 0.875). Monotonic, no dips, no spikes.
- **Field data validation:** 250yd dip corrected +4.3%, 150yd spike corrected -3.4%, other 4 points moved <±1.7%
- **250yd anomaly and 150yd spike confirmed as field measurement noise, not real bullet physics**
- **Practical impact of curve change:** 0.003 mils between truing points, 0.084 mils at 300yd extrapolation. Invisible at the match. The value is a defensible foundation ("Doppler lab data" vs "six groups one afternoon")

---

## KESTREL MATCH INSTRUCTIONS

### Setup — Two Profiles
- **Profile 1 "SHORT":** G7, BC 0.078, trued at ~125yd. Use for targets 50-150yd. Validated at competitions.
- **Profile 2 "LONG":** G1, trued at 225yd. Enter real MV, start at G1 0.145, adjust BC until 225yd prediction matches dial-up. Will land around 0.50-0.55. Don't touch MV.

### Why G7 Short / G1 Long
- G7's error shape in the high-Mach zone (Mach 0.85-0.95) is flatter than G1's — a single BC absorbs it better
- G1 becomes progressively less wrong as bullet slows into deep subsonic (past 200yd)
- Optimal split: G7 where G1 is most wrong, G1 where it's least wrong

### Match-Day Kestrel Usage
- **Below 150yd:** Trust Kestrel SHORT profile (validated at matches)
- **150-275yd:** Trust the app (TM22 mid-range ±0.06-0.12 MIL cross-condition)
- **Above 275yd:** Average app and Kestrel LONG, lean toward app
- If they disagree >0.3 MIL at mid-range: re-check zero, re-check DA entry

### Re-true Rules
- **App:** Almost never needs re-truing for weather. Just update temp/DA in HUD — Mach-indexed corrections auto-adjust. Re-true only for physical changes: new ammo lot, new scope mount, barrel wear.
- **Kestrel:** Needs re-true when DA shifts >2000ft or temp changes >30°F from truing day (G1/G7 BC was trued at a specific Mach at a specific distance — conditions change the Mach, making the BC wrong).
- **New ammo lot:** Re-chrono MV and re-true everything.
- **This is TM22's core advantage:** app handles condition changes automatically, Kestrel can't.

### Why Kestrel Scaling Has Limits
- When you trued G1 at 225yd on one day, you found the BC that works at that day's terminal Mach at 225yd
- On a different DA day, the terminal Mach at 225yd shifts by ~±0.010 per ±1500ft DA
- G1's error at the new Mach is slightly different → trued BC is slightly wrong
- Effect: 0.1-0.2 MIL at 250-300yd for typical seasonal variation. Competitive but not perfect.
- **TM22 eliminates this entirely** — Mach-indexed corrections track automatically

---

## KEY FORMULAS

```
// Stepped BC lookup (engine)
lookupSteppedBc(dist, points) → interpolated BC at distance

// BC solver (binary search)
solveBcFromDialUp(dist, dialMils, mv, zeroD, scopeH, baseBc) → {bc, mach}

// TM22 factor (informational only)
factor = solved_BC / base_BC

// Confidence weight
weight = sqrt(shots) / groupMOA

// Mach-band merge threshold
0.015 Mach (~17fps terminal velocity shift)

// Minimum truing distance
minDist = 2.5 × zeroDistance

// Wind drift (lag-time / Didion)
lag = TOF - distance / MV
drift = crosswind_fps × lag × 12

// Crossing angle
relAngle = windFromDeg - targetBearingDeg + 180
crosswind = windSpeed × sin(relAngle)

// MV temperature correction
adjustedMV = baseMV + (temp - 59) × 0.8

// Scope tracking correction
corrMils = dialMils / trackingFactor

// DA affects drag force, not TM22 factors
drag ∝ rhoRatio × G1_Cd(mach) / (baseBc × factor)
// factor = bullet property (shape), rhoRatio = atmosphere property (density)
```

---

## DATA ARCHITECTURE (V2)

### Rifle Profile
```js
{
  id, name, zeroDistance: 50, scopeHeight: 2.6, barrelLength: 20,
  twistRate: 16, clickSize, turretUnit: 'MIL'|'MOA',
  scopeTrackingCorrection: 1.0  // 1.0 = perfect, 0.95 = 95%
}
```

### Ammo Profile
```js
{
  id, profileName,
  manufacturer: 'SK', product: 'Rifle Match',
  lotNumber: '844',
  muzzleVelocity: 1056, sd: 8, es: 28, bulletWeight: 40,
  bc: 0.135, bcModel: 'G1',  // locked
  groupSize, groupUnit: 'MOA'|'MIL'|'IN'
}
```

### Truing Point (per-point full snapshot)
```js
{
  id, timestamp,  // ISO 8601
  sessionId,      // groups collection mode points
  sessionNote,    // "Session 2 - Feb 8 dense collection"
  distance, dialUp, dialUpUnit: 'MIL'|'MOA'|'IN',
  shots, groupSize, groupUnit,
  solvedBc, factor, mach,  // factor = solvedBc / baseBc
  lowConf: boolean,        // true if dist < 2.5 × zero
  conditions: {
    temp, da, mv, profileMV,
    trackingCorr, zeroDistance, scopeHeight, barrelLength,
    bcModel: 'G1', baseBc, bulletWeight
  }
}
```

---


## DOCUMENTS CREATED

| Document | Contents |
|----------|----------|
| V3 JSX (artifact) | Current app with TM22 v2 reference curve engine |
| V2 JSX (artifact) | Previous stepped-BC-only version |
| V1 JSX (artifact) | Archived full app, ~11,400 lines |
| Handoff (this doc) | Complete system state |
| TM22 Technical Findings (MD) | 8 findings + roadmap + 2×2 experimental design |
| TM22 Technical Paper (PDF) | 12 sections, 10 tables, formulas — for ballistics community |
| TM22 Range Protocol V1 (PDF) | 5-page data collection sheets (2×2 factorial) |
| TM22 Match Prep Guide (PDF) | 6 pages: setup, truing, match-day workflow, CZD/CZT sheets |
| TM22 Curve Analysis (JSX) | Interactive factor/BC curves from Session 2 |
| TM22 Curve Overlay (JSX) | Dual-axis overlay of factor + BC curves |
| TM22 vs AB CDM (JSX) | 5 AB CDM curves overlaid against TM22 empirical data |
| TM22 SK Extraction (JSX) | Field vs lab comparison, upgraded curve, new reference table |
| Wind Bracket (JSX) | Statistical first-round hit probability calculator |
| Wind HUD (JSX) | Real-time DOPE with simulated Calypso wind stream |
| Strelok Salvage (ZIP, 11 files) | BLE protocols, DB schemas, 1895 reticles, targets, features |
| Sig BDX Salvage (ZIP + MD) | BLE protocol, GATT UUIDs, command set, message types |
| TM22 DOPE Display Prototype (ZIP) | Parts list, firmware, build guide |
| TM22 DOPE V1 Revised (ZIP) | Long-range BLE, LiPo, BME280 weather, compass mount |
| TM22 DOPE V1 Final Build-Once (ZIP) | Full sensor suite: weather + IMU + mic + LED |
| TM22 DOPE V1 Delta (ZIP) | Acoustic shot detection + ocular LED updates |
| Bolt Build Guide (MD) | 8 work groups with exact Bolt prompts, build order, verification steps, critical rules |
| Session 34 Findings (MD) | Bolt corrections: MV double-counting fix, single BC solver, air density formula, field validation, error analysis |
| TM22 Capacitor Build (ZIP) | Capacitor Android project: www/index.html + BLE + persistence + build guide |
| Bolt Rebuild Plan (MD) | 7 task groups with exact prompts |
| Native Kotlin Rewrite Plan (DOCX) | Architecture doc, 10-12 week timeline |

---

## STRELOK APK SALVAGE (Feb 10)

Strelok Pro APK decompiled (strings + SQL extraction, no full decompile). 11 files extracted:

### High-Value Finds
- **Kestrel 5x00 BLE UUIDs:** 67 proprietary characteristics under `ffffffff-c446-4c4f-*` base. Which UUID = which measurement needs nRF Connect mapping.
- **Calypso BLE:** Standard Environmental Sensing Service (0x181A). UUIDs: 0x2A6E (temp), 0x2A6D (pressure), 0x2A6F (humidity), 0x2A72 (wind speed), 0x2A73 (wind direction). Trivial integration.
- **Database schemas:** All SQL tables. 5-point stepped BC (validates TM22 architecture).
- **Powder temp model:** 5-point MV/temperature table + linear coefficient. Worth implementing.
- **Zero atmosphere storage:** Tracks conditions at zero separately from current conditions.
- **1,895 reticles across 64 brands** (CSV), including 24 rimfire-specific and 20 Athlon entries.
- **Target viewer architecture:** Virtual scope overlay, FFP/SFP scaling, NRL22 target table, moving target lead.
- **13 BLE drivers total:** 6 weather, 4 rangefinders (Athlon, MTC, NTC, Vectronix — NO Sig), 2 thermal, 1 unknown.
- **Kestrel bidirectional profile push:** Pushes G1/G7 profiles TO Kestrel — but Kestrel re-solves with AB engine, so TM22 curves can't be pushed.
- **Field tools:** MildotRanger (camera ranging), HUD bench mode, GPS range cards.

### Strelok Context
- Developer (Borisov) Russian, US State Department sanctions list. App removed from Apple App Store.

---

## SIG BDX PROTOCOL SALVAGE (Feb 11)

Sig BDX APK v346 decompiled. Full BLE command architecture extracted.

- **Two BLE transports:** Legacy: FFF1 (write) / FFF2 (notify). K-Series (3K+): ISSC Transparent UART (49535343-xxx).
- **Six RX message types:** RangeMessage (distance + angle), SolutionMessage (ignore — TM22 replaces), EnvironmentGetMessage (temp/pressure/humidity/altitude), WindMeterMessage, NackMessage, NoOpMessage.
- **17+ TX commands:** sendBLEGetRange(), sendGetEnvironment(), profile push, bonding, calibration.
- **Missing:** Exact byte-level packet format. Needs nRF Connect capture from user's Kilo 3K.
- **User's hardware:** Sig Kilo 3K — BLE 5.x, multipoint, BDX-X mode (designed for third-party), onboard environmental sensors.
- **Other finds:** OpenWeatherMap Pro API key, AB bullet library API endpoint, firmware binaries for all Kilo models, Echo3 thermal runs ESP32.

---

## WIND BRACKET PROBABILITY SYSTEM (Feb 11 — NOVEL FEATURE)

### Concept
Given target size, distance, system precision (base MOA), and two wind estimates (W1/W2) with frequency weighting, compute the hold that maximizes first-round hit probability.

### Math
- Convolution of Gaussian system dispersion with uniform wind uncertainty
- Numerical integration across target geometry (angular size at distance)
- Compares: naive hold (midpoint), optimal hold, and single-condition holds
- Key insight: optimal hold shifts toward dominant wind condition on small targets
- Can recommend "wait for lull" when blended P(hit) is poor but single-condition P(hit) is high

### Wind HUD (Real-Time)
- Simulated Calypso at 1Hz, rolling window (30/60/120s), k-means clustering (k=2)
- Auto-detects W1/W2 and frequency ratio from live data
- Live DOPE card with confidence indicators per target (green/yellow/red)
- Wind deflection currently linear placeholder — will be replaced by TM22 solver

---

## E-DOPE CARD & STAGE SEQUENCE MODE

- K&M E-Dope: e-paper, NFC from phone, Picatinny mount, sunlight readable, battery-less
- TM22 renders DOPE → NFC tap writes image → 2 seconds
- **Stage sequence mode:** Shows ONE target at a time (large text), auto-advances on shot detection
- Stage builder defines engagement ORDER with shots per target and position labels
- Position transitions labeled, makeup shot button, shot counter

---

## TM22 DOPE DISPLAY HARDWARE (Feb 11)

### Build-Once Card Spec (V1 Final)
| Component | Part | Cost | Function |
|-----------|------|------|----------|
| MCU | XIAO ESP32-C3 (U.FL antenna) | $5 | Brain + long-range BLE (50-150m) |
| Display | 2.9" e-paper | $14 | Sunlight-readable DOPE |
| Weather | BME280 | $3.50 | Temp/pressure/humidity (replaces Kestrel weather) |
| IMU | LSM6DSO | $6 | Cant ±0.2°, stability, muzzle trace (replaces SG Pulse) |
| Shot detect | MAX4466 MEMS mic | $1.50 | Acoustic, adjustable sensitivity |
| Cant LED | WS2812B RGB (ocular mount) | $0.50 | Green=level, red=right cant, blue=left cant |
| Power | 500mAh LiPo + USB-C | $4 | 50-80 match days per charge |
| **Card total** | | **~$35** | |

### Compass Mount (Picatinny, per-rifle)
- LIS3MDL magnetometer + 4 pogo pins + PCB → ~$17 prototype
- Card seats on mount, pogo pins bridge I2C for compass data
- One mount per rifle, card moves between them

### Why Acoustic Shot Detection
- SG Pulse (user's current device) uses acoustic, not accelerometer
- Competition .22LR rifles up to 30 lbs — zero recoil at rail
- MEMS mic hears muzzle blast regardless of weight, even suppressed
- Rise time separates shot (<1ms) from handling (50-200ms)

### IMU Bonus Functions
- Anti-cant: ±0.2° (matches user's SG Pulse setting)
- Muzzle tracking: 6.66 kHz captures recoil waveform (training data)
- Inclinometer: angle shot cosine correction
- Military variant: no LED cable (light signature), cant on e-paper only

### Future V2: Integrated Ultrasonic Anemometer
- 4× 40kHz transducers + ESP32 timers (12.5ns resolution) → replaces Calypso
- BOM adds ~$4. Not V1 — prove software first, build wind sensor later.

### Replaces ~$2,180 in Commercial Gear
Kestrel ($450) + Calypso ($349) + SG Pulse ($150) + E-Dope ($130) + ...

---

## BLE INTEGRATION PLAN

### Phone as Hub — Three Simultaneous BLE Peripherals
1. **TM22 Card** — weather + cant + shots + display output
2. **Sig Kilo 3K** — range + angle + backup weather
3. **Calypso Mini** — wind speed + direction (until V2 card integrates wind)

### Integration Status
| Device | Protocol Status | Integration Effort |
|--------|----------------|-------------------|
| Calypso | Standard BLE 0x181A, fully documented | One afternoon |
| Kestrel 5700 | 67 UUIDs extracted, mapping needed | nRF Connect session |
| Sig Kilo 3K | Protocol architecture mapped, byte format needed | nRF Connect session |
| Athlon rangefinder | Driver found in Strelok | Future |

### Android Requirements
- Permissions: BLUETOOTH, BLUETOOTH_CONNECT, BLUETOOTH_SCAN, ACCESS_FINE_LOCATION
- Foreground service with persistent notification
- No Play Store policy issues

---

## PRODUCT VISION & MONETIZATION (Feb 11)

### What This Is
An **NRL22 competition platform** — not just a ballistic calculator. Includes: TM22 solver, wind bracket probability, stage builder with engagement sequencing, shot detection with auto-advance, shooter analytics, live BLE weather, E-Dope/hardware display output, and camera group analysis.

### Market
- 8,000-12,000 active competitive rimfire shooters in US (PRS Rimfire 4,000+, NRL22 150+ clubs)
- Wind bracket system works for any caliber → expands to all precision rifle (~13,000+ PRS competitors)
- Fastest-growing segment — PRS Rimfire didn't exist before 2021

### Monetization (Ranked)
1. **Content (YouTube):** 10-episode series. Build credibility + audience. Nobody doing rimfire at this depth.
2. **Data licensing:** TM22 curves to AB/Kestrel, Revic, SIG. Per-unit royalty. Passive.
3. **App subscription:** $9.99/month Pro tier. 2,000 subs = $240K ARR at scale.
4. **Hardware:** DOPE display $150-200 retail, $25-35 BOM, 75-85% margin.
5. **Defense (long term):** Fire control system kit. $300 BOM → $3,000-5,000 system sale.

### Patent Consideration
- Wind bracket probability applied to fire control = novel
- Live anemometer → clustering → probabilistic optimization → rifle-mounted display
- Provisional: $320, 12 months protection

---

## BUILD STRATEGY (Revised Feb 11)

- **Claude = R&D bench.** Design and prove each module as working code.
- **Bolt = production line.** Bolt writes every line from proven logic. Owns and understands the code.
- **Temp Android app** for field-proving. Throwaway code. Teaches what real app needs.
- **Bolt app is the real product** — full NRL22 competition platform, modular.

### Module Roadmap
| # | Module | Status |
|---|--------|--------|
| 1 | TM22 solver + stepped BC | DONE |
| 2 | Multi-rifle/ammo profiles | DONE |
| 3 | Truing + collection mode | DONE |
| 4 | Wind bracket probability | PROVEN (JSX) |
| 5 | Wind HUD (live Calypso sim) | PROVEN (JSX) |
| 6 | Data persistence (localStorage) | DONE — in Capacitor build |
| 7 | BLE service layer | DONE — DOPE card, Calypso, Kestrel, Sig Kilo UUIDs + scan/connect/read/write/notify |
| 8 | BLE panel + HUD auto-fill | DONE — weather auto-populates temp/DA from Calypso/Kestrel |
| 9 | Capacitor Android wrapper | DONE — ZIP ready, user builds locally |
| 10 | Stage builder (engagement order) | DESIGNED |
| 11 | BLE weather (Kestrel/BME280) | SERVICE DONE — needs nRF Connect UUID mapping |
| 12 | BLE rangefinder (Sig Kilo 3K) | SERVICE DONE — needs nRF Connect byte format capture |
| 13 | BLE wind (Calypso) | SERVICE DONE — standard BLE, parser implemented |
| 14 | E-Dope NFC output | DESIGNED |
| 15 | Hardware DOPE card | SPECCED — parts to order |
| 16 | DOPE packet builder + sender | DONE — binary format matching ESP32 firmware |
| 17 | Shot detect + auto-advance | SPECCED (firmware) |
| 18 | Cant/level (IMU + LED) | SPECCED (firmware) |
| 19 | Group analysis (camera) | REFERENCE — Range Buddy APK uploaded, extraction interrupted |
| 20 | Shooter analytics/metrics | CONCEPT |
| 21 | Powder temp compensation | SCHEMA (from Strelok) |
| 22 | Virtual target viewer | ARCHITECTURE (from Strelok) |
| 23 | GPS range cards | CONCEPT |
| 24 | Camera wind reading | FUTURE |
| 25 | Ultrasonic anemometer (V2 card) | FUTURE |

---

## SESSION LOG

| Session | Date | Work Done |
|---------|------|-----------|
| 1-4 | Feb 5-6 | V1: Bug fixes, G7 engine, geometry, uncertainty, wind, stage builder |
| 5-6 | Feb 6 | V1: Wind physics 70/30, bracket model, Bolt migration, Capacitor |
| 7 | Feb 7 | React Native attempt (FAILED), architecture decision |
| 8 | Feb 7 | V2 prototype: HUD, Truing, Profiles, all units, wind |
| 9 | Feb 7 | BC solver debug, G1-only lock, manufacturer research |
| 10 | Feb 7-8 | RA4 research (dead end), TM22 concept, AB DSF analysis |
| 11 | Feb 8 | TM22 Mach-variable engine built → FAILED → reverted to stepped BC |
| 12 | Feb 8 | Confidence weighting, Mach-band merge, min distance gate |
| 13 | Feb 8 | Short-range problem analysis, collection mode, protocol PDF |
| 14 | Feb 8-9 | Technical paper, scope height fix (2.6"), Session 2 data entry |
| 15 | Feb 9 | Curve analysis, cross-session comparison, next session planning |
| 16 | Feb 9-10 | Short-range G7 cross-check, peak shift discovery, two-pass design |
| 17 | Feb 10 | Session 3 data, cross-condition validation, chrono discrepancy |
| 18 | Feb 10 | TM22 reference curve with scaffolding (V3), 77% noise reduction |
| 19 | Feb 10 | Match prep: CZD/CZT profiles, Kestrel instructions |
| 20 | Feb 10 | AB CDM extraction (5 ammos), two TM22 families, comparison tool |
| 21 | Feb 10 | TM22 v2 — AB lab curve replaces scaffolding, polynomial smoothing |
| 22 | Feb 10 | Strelok APK salvage — 11 files, BLE, 1895 reticles, DB schemas |
| 23 | Feb 10-11 | Kestrel/Calypso/BLE integration research |
| 24 | Feb 11 | Wind bracket probability system (novel) — math proven, JSX |
| 25 | Feb 11 | Wind HUD — real-time Calypso sim, live DOPE, JSX |
| 26 | Feb 11 | E-Dope integration, stage sequence mode designed |
| 27 | Feb 11 | DOPE display hardware — ESP32 + e-paper spec (3 iterations) |
| 28 | Feb 11 | Shot detection, IMU cant, ocular LED — build-once spec |
| 29 | Feb 11 | Sig BDX APK salvage — full protocol extraction |
| 30 | Feb 11 | Monetization, YouTube series, defense angle, product vision |
| 31 | Feb 11 | Range Buddy APK — group analysis reference (extraction interrupted by context limit) |
| 32 | Feb 12-13 | **Handoff rebuild chat** — compacted mid-session. Merged two transcripts. Added Guide tab (app + Kestrel unified truing, match-day workflow, theory). Fixed re-true rules (app doesn't need re-true for weather). Built Capacitor Android project with BLE service layer (DOPE card, Calypso, Kestrel, Sig Kilo), localStorage persistence, BLE tab, HUD auto-fill from BLE weather. |
| 33 | Feb 16 | **Bolt Build Guide** — 8 work groups with exact prompts, build order, verification steps, critical rules. Complete handoff rebuilt. APK build attempted (Gradle). |
| 34 | Feb 17 | **Single BC solver + field validation.** Session 3 data analyzed (70°F, DA 1007, chrono'd 1068). Found MV double-counting bug (0.35→0.18 MIL). Built single TM22 BC solver replacing stepped BC (0.18→0.13 MIL avg). Fixed HUD state persistence across tabs. Blocked negative dial-ups. Found wrong air density formula in Bolt Guide. Temp coefficient measured 0.2 fps/°F for SK lot 844 (vs 0.8 default). L-M Cd correction tested, overfits — deferred. Error analysis: curve shape 40%, MV uncertainty 30%, measurement noise 30%. Session 34 findings document created with all Bolt corrections. |
| **Total** | | **~800+ messages across 8 chats** |

---

## INSTRUCTIONS FOR CONTINUING

### Current Files
- **V3 JSX** (`ballistic-logger-v3.jsx`): Working app artifact for Claude
- **Capacitor build** (`tm22-capacitor.zip`): Complete Android project — unzip, npm install, npx cap add android, npx cap sync android, npx cap run android. Location: C:\projects\tm22-capacitor
- **Bolt Build Guide** (`BOLT-BUILD-GUIDE.md`): 8 work groups with exact prompts for Bolt to rebuild from scratch
- **This handoff** (`HANDOFF-22LR-BALLISTIC-CALC.md`): Complete system state

### If continuing V3 in Claude:
1. V3 has TM22 v2 reference curve (AB SK Match CDM, polynomial-smoothed, 59 Cd points)
2. Engine: 85% Doppler lab data, ~15% edge scaffolding
3. Solver noise 77% lower than G1
4. Scope height: CZD **2.6"**, CZT **~2.0"** (verify)
5. Ammo: **SK Rifle Match lot 844** — Caldwell chrono baseline 1056fps
6. Test against Session 2: 75=0.8, 150=4.35, 200=7.30, 275=12.45

### If rebuilding in Bolt:
1. **READ SESSION-34-FINDINGS.md FIRST** — contains critical corrections to all work groups
2. Feed the Bolt Build Guide work groups one at a time, in order (WG1→WG2→WG4→WG3→WG5→WG6→WG7)
3. Apply Session 34 corrections to each WG as you go (air density formula, single BC solver, chrono MV bypass, negative rejection, HUD state lift)
4. Verify each work group before moving to the next
5. Use Opus 4.6 in Bolt for best results
6. The engine math (WG1) MUST be exact — copy data tables verbatim, use ISA power law (NOT exponential)
7. Paste the TM22_TABLE and G1_TABLE arrays directly from this handoff or the Bolt guide
8. After WG7, you have a working APK with identical functionality to the Capacitor build

### What the user cares about:
- ±0.1 mil accuracy, one profile 50-300yd, scientific approach
- **NRL22 competition platform** — solver + wind bracket + stage builder + hardware
- TM22 as publishable model and potential business
- BLE integration: Kestrel, Calypso, Sig Kilo 3K
- Hardware DOPE display prototype
- Claude = R&D, Bolt = production. Handoff must be comprehensive.
- Semi-pro NRL22/PRS22 shooter, nationals-level. Wife (CZT) also competing.
- **Creativity: "dangerous human"** — sees gaps across $2K+ of commercial products

### Immediate (Pre-Match ~Feb 21):
1. Pre-match truing: CZD + CZT fresh-trued on Athlon
2. nRF Connect recon: Kestrel 5700 weather UUIDs + Sig Kilo 3K range packets

### Post-Match:
3. Match data analysis, assess TM22 real-world accuracy
4. Order + build DOPE display prototype
5. Finish Range Buddy group analysis extraction
6. Begin Bolt module builds (solver first, then wind bracket)
7. Strelok SDK upload → decoded Kestrel characteristics
8. YouTube content if user decides on content path

### Hardware (Post-Match):
9. Build DOPE V1: ESP32 + e-paper + BME280
10. BLE sessions: Kestrel, Kilo 3K, Calypso
11. Integrate weather into temp Android app
12. Add shot detection + IMU + LED
13. Test complete system at matches with crew
