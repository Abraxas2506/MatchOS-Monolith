# Session 34 Findings — Bolt App Corrections

**Date:** February 17, 2026
**Context:** Three-session field validation (S1/S2/S3), engine bugs found and fixed, solver architecture changed
**Applies to:** BOLT-BUILD-GUIDE.md (all WGs affected), HANDOFF-22LR-BALLISTIC-CALC.md

---

## CRITICAL BUGS FOUND

### Bug 1: MV Double-Counting (SEVERITY: HIGH — caused 0.35 MIL avg error)

**What happened:** User chrono'd 1068 fps on a 70°F day and entered it in the HUD MV Override field. The engine then applied temp compensation on top: `adjustedMV = 1068 + (70 - 59) × 0.8 = 1076.8 fps`. The chrono already measured the warm powder's actual velocity — applying temp comp again added ~9fps of phantom speed, making the solver under-predict drop at every distance.

**Root cause:** `calculateDrop()` always applies `adjustedMV = muzzleVelocity + (temperature - 59) * 0.8` regardless of whether the MV came from a chrono (already temp-adjusted) or a profile baseline (needs temp comp).

**Fix:** When `hasChrono` is true (user entered MV Override), pass `temperature = 59` to `calculateDrop()` so the temp comp term zeroes out. The chrono'd MV is used as-is.

```javascript
// In HUD solver:
const solveTemp = hasChrono ? 59 : parseFloat(hs.temp);
calculateDrop({...params, temperature: solveTemp}, cdTable);
```

**Impact of fix:** Average error dropped from 0.35 → 0.18 MIL across all S3 predictions.

### Bug 2: Air Density Formula Wrong in Bolt Guide (SEVERITY: HIGH)

**The Bolt Build Guide WG1 says:**
```
rhoRatio = exp(-densityAltitude / 34000)
```

**It should be (ISA power law):**
```
rhoRatio = Math.pow(1 - densityAltitude / 145442, 4.256)
```

The exponential approximation diverges at high DA. The ISA power law is what's in the working Capacitor app and what Bolt validated against. **This must be corrected in WG1 before feeding to Bolt.**

### Bug 3: HUD State Lost on Tab Switch (SEVERITY: MEDIUM)

**What happened:** HUD state (temp, DA, MV, wind, range) was local React state inside the HudTab component. When the user switched to Truing and back, HudTab unmounted and all environmental inputs were lost.

**Fix:** Lift HUD state into App component, pass down as props, persist in localStorage alongside rifles/ammo/truingData.

### Bug 4: Negative Dial-Ups Accepted (SEVERITY: LOW)

**What happened:** User entered -6 MIL at 75yd (typo, meant 0.6). Solver clamped to BC 0.5000, poisoned the factor curve with ×3.704 entries.

**Fix:** Reject `rawDial <= 0` in addPt (error message) and saveCollection (skip silently). Also clamp at solver level: `if (dialUpMils <= 0) return { bc: 0.135, clamped: "negative" }`.

---

## ARCHITECTURE CHANGE: Single TM22 BC Replaces Stepped BC

### Why Stepped BC Failed Cross-Condition

The stepped BC engine solves an independent BC at each truing distance, then interpolates by distance to predict new distances. This works perfectly for round-trip validation (predict the same conditions you trued under) but breaks across conditions because:

- Each distance-indexed BC gets applied at the wrong Mach when DA/temp changes
- The Mach at 225yd on a 70°F day (M0.78) ≠ the Mach at 225yd on a 55°F day (M0.77)
- This is exactly the problem the TM22 Mach-indexed curve was supposed to solve, but stepped BC defeats it

### Single BC Architecture

One BC value scales the entire TM22 Cd curve. Solved by minimizing total squared error across ALL truing points simultaneously, each evaluated under its OWN recorded conditions (temp, DA, MV).

**`solveSingleTM22Bc(points)`** — new function added to engine:

1. Filter truing points that have full condition snapshots and positive corrMils
2. For each point, determine if MV was chrono'd (mv ≠ profileMV by >3fps → temp=59) or estimated (use recorded temp)
3. Weight: high-conf points × 1.0, low-conf (< 2.5× zero) × 0.5
4. Golden section search over BC range [0.01, 0.15] minimizing weighted sum of squared errors
5. Return `{ bc, avgError, maxError, numPoints, residuals[] }`

### Performance Comparison (16 points, 3 sessions, 1335ft DA swing)

| Method | Avg Error | Max Error | Params |
|--------|-----------|-----------|--------|
| Old stepped BC + double-counted MV | 0.350 MIL | 0.450 MIL | 6 BCs |
| Fixed stepped BC + chrono fix | 0.178 MIL | 0.278 MIL | 6 BCs |
| **Single TM22 BC (all data)** | **0.129 MIL** | **0.232 MIL** | **1** |

Cross-validation (train S1+S2 → predict S3): avg 0.129, max 0.232

### Short-Range Impact (50–175yd)

Changing from stepped BC to single BC moves predictions by <0.01 MIL at 100yd and 0.013 MIL at 175yd. **BC barely matters under 125yd** — swinging BC from 0.040 to 0.070 (75% change) only moves the prediction 0.18 MIL at 100yd. The bullet is still fast, drag hasn't accumulated, geometry from the 50yd zero dominates.

All short-range KZ hits are preserved. The single BC improves long-range without hurting short-range.

---

## FIELD DATA — SESSION 3 (New)

**Date:** Feb 16, 2026
**Conditions:** 70°F, DA 1007, chrono'd 1068 fps (Athlon)
**Rifle:** CZD, 50yd zero, scope height 2.6"
**Ammo:** SK Rifle Match, lot 844

| Distance | Dial-Up | Solved BC | Velocity | Mach |
|----------|---------|-----------|----------|------|
| 75yd | 0.8 MIL | 0.0311 (LOW) | 951fps | 0.84 |
| 100yd | 1.8 MIL | 0.0416 (LOW) | 952fps | 0.84 |
| 150yd | 4.1 MIL | 0.0543 | 937fps | 0.83 |
| 175yd | 5.5 MIL | 0.0512 | 912fps | 0.81 |
| 200yd | 7.1 MIL | 0.0461 | 878fps | 0.78 |
| 225yd | 8.7 MIL | 0.0452 | 855fps | 0.76 |
| 250yd | 9.9 MIL | 0.0525 | 863fps | 0.76 |
| 275yd | 11.7 MIL | 0.0497 | 836fps | 0.74 |
| 302yd | 13.7 MIL | 0.0482 | 812fps | 0.72 |

**Single TM22 BC from all 3 sessions: 0.0534**

---

## TEMP COEFFICIENT FINDING

The 0.8 fps/°F default is too high for SK Rifle Match lot 844.

**Evidence:**
- S1 Caldwell chrono: 1056 fps at 55°F
- S3 Athlon chrono: 1068 fps at 70°F
- Athlon reads ~9fps high → Caldwell-equivalent: ~1059 fps
- Actual change: +3fps over +15°F = **~0.2 fps/°F**

The app uses 0.8 fps/°F (generic industry number). This caused over-compensation when using estimated MV.

**Current fix:** Chrono'd MV bypasses temp comp entirely.
**Future fix:** Per-ammo configurable temp coefficient field in ammo profile, default 0.8.

---

## REMAINING ERROR ANALYSIS

### Where ±0.13 MIL Average Comes From

1. **Curve shape tilt (~40% of error):** Residuals are systematic — positive at Mach 0.82–0.84 (125–150yd), negative at Mach 0.73 (302yd). The AB lab curve is slightly too draggy at upper Mach, slightly too slippery at lower Mach for this specific bullet. A Cd tilt correction helps on all-data fit but overfits with only 9 training points.

2. **MV uncertainty (~30% of error):** S1/S2 weren't chrono'd. At 302yd, 1fps = 0.017 MIL. If real MV was off by 5fps, that's 0.085 MIL — most of the error budget.

3. **Measurement noise (~30% of error):** At 150yd, the BC spread across sessions is 22.4%. At 250yd it's 3.6%. Short-range points are noisy because total correction is small (~1.4" at 150yd from 50yd zero). A half-click turret error swings the solved BC by 25%.

### Path to ±0.10 MIL

Cannot be achieved by solver improvements alone. Requires better input data:

1. **Always chrono on the day** — eliminates MV uncertainty (30% of error)
2. **3+ groups per distance, average dial-ups** — reduces measurement noise
3. **Drop ≤125yd from solver input** — too noisy to help, doesn't hurt short-range predictions
4. **5+ sessions across >2000ft DA spread** — constrains the BC estimate and eventually enables Cd tilt correction without overfitting

---

## BOLT BUILD GUIDE CORRECTIONS

### WG1 Corrections (Engine)

**1. Air density formula — MUST FIX:**
```
// WRONG (in current guide):
rhoRatio = exp(-densityAltitude / 34000)

// CORRECT (ISA power law):
rhoRatio = Math.pow(1 - densityAltitude / 145442, 4.256)
```

**2. Add `solveSingleTM22Bc(points)` function:**
```javascript
function solveSingleTM22Bc(points) {
  // Filter valid truing points with condition snapshots
  const evalPts = points.filter(p => p.conditions && p.corrMils > 0 && p.distance > 0).map(p => ({
    dist: p.distance,
    dial: p.corrMils,
    mv: p.conditions.mv || p.conditions.profileMV || 1056,
    // Chrono'd MV (mv ≠ profileMV) → temp=59 to disable comp
    temp: (p.conditions.mv && p.conditions.profileMV && 
           Math.abs(p.conditions.mv - p.conditions.profileMV) > 3) ? 59 : (p.conditions.temp || 59),
    da: p.conditions.da || 0,
    zero: p.conditions.zeroDistance || 50,
    sh: p.conditions.scopeHeight || 2.6,
    lowConf: p.lowConf
  }));
  if (evalPts.length === 0) return null;

  const cost = (bc) => {
    let sum = 0;
    for (const p of evalPts) {
      const r = calculateDrop({
        distance: p.dist, muzzleVelocity: p.mv, bc,
        zeroRange: p.zero, scopeHeight: p.sh,
        temperature: p.temp, densityAltitude: p.da
      }, TM22_TABLE);
      const w = p.lowConf ? 0.5 : 1.0;
      sum += w * (r.dropMils - p.dial) ** 2;
    }
    return sum;
  };

  // Golden section search
  let lo = 0.01, hi = 0.15;
  for (let i = 0; i < 80; i++) {
    const m1 = lo + (hi - lo) * 0.382, m2 = lo + (hi - lo) * 0.618;
    if (cost(m1) < cost(m2)) hi = m2; else lo = m1;
  }
  const bc = (lo + hi) / 2;

  // Compute residuals
  let sumErr = 0, maxErr = 0;
  const resid = evalPts.map(p => {
    const r = calculateDrop({
      distance: p.dist, muzzleVelocity: p.mv, bc,
      zeroRange: p.zero, scopeHeight: p.sh,
      temperature: p.temp, densityAltitude: p.da
    }, TM22_TABLE);
    const err = r.dropMils - p.dial;
    sumErr += Math.abs(err);
    if (Math.abs(err) > maxErr) maxErr = Math.abs(err);
    return { dist: p.dist, predicted: r.dropMils, actual: p.dial, error: err };
  });

  return {
    bc, avgError: sumErr / evalPts.length, maxError: maxErr,
    numPoints: evalPts.length, residuals: resid
  };
}
```

**3. Keep `lookupSteppedBc` as legacy** — used by the factor curve display in Truing tab, not for predictions.

### WG2 Corrections (Profiles)

**Add to ammo profile fields:**
- `tempCoeff` — MV temperature coefficient, fps per °F, default 0.8. Display in ammo form as "MV Temp Sensitivity" with helper text "0.8 = industry default, measure yours across sessions"

### WG3 Corrections (Truing)

**1. Reject negative dial-ups:**
```javascript
if (rawDial <= 0) { setErr("Dial-up must be positive"); return; }
```

**2. Add Single BC display panel:**
After the factor curve display, show a green-bordered panel:
- "SINGLE TM22 BC" header
- Solved BC value (large, 4 decimal places)
- Point count, session count (count unique DA values)
- Avg error, max error in MIL
- Per-point residual grid: Dist | Actual | Predicted | Error (color-coded)

**3. Store conditions with every truing point (already in V3, verify in Bolt):**
Each point must snapshot: temp, DA, MV used, profileMV, trackingCorr, zeroDistance, scopeHeight, bcModel, baseBc

### WG4 Corrections (HUD)

**1. Replace solver engine:**
```javascript
// OLD: distance-interpolated stepped BC
const bc = st ? (lookupSteppedBc(dist, st) || baseBc) : baseBc;

// NEW: single TM22 BC from all truing data
const singleBcResult = solveSingleTM22Bc(st); // memoize with useMemo
const bc = singleBcResult ? singleBcResult.bc : baseBc;
```

**2. Two MV fields:**
- "Chrono'd MV" — editable input, green border when populated
- "Est. MV" — read-only display: `profileMV + (temp - 59) × tempCoeff`
- Status line: green "USING: Chrono'd 1068 fps — no temp adjustment applied" or yellow "USING: Est. 1061 fps (profile 1056 +4.0 temp)"

**3. Chrono'd MV disables temp comp:**
```javascript
const solveTemp = hasChrono ? 59 : parseFloat(hs.temp);
```

**4. Badge shows:** `BC 0.053 ±0.13` instead of old `TM22 ×16`

**5. Solver info panel:** Below elevation display, show: `TM22 BC 0.0534 from 16 pts | avg ±0.13 max ±0.23`

**6. Lift HUD state to App level** — persist temp, DA, MV, wind, range in localStorage. Pass as props to HudTab.

### WG5 Corrections (Guide)

**Update Truing section:**
- Add: "Always chrono on the day. Enter chrono'd MV in the HUD — the app will use it directly without temp adjustment."
- Add: "Skip distances ≤125yd for truing. The physics is too insensitive at short range to extract a reliable BC from a 50yd zero. Your short-range hits won't be affected."
- Add: "The app now solves a single TM22 BC across all your truing data. More sessions under different conditions = tighter BC estimate."

**Update Match Day section:**
- Add: "If you chrono'd: enter actual MV, the app uses it as-is"
- Add: "If you didn't chrono: leave MV blank, app estimates from profile + temp"

**Update Re-True Rules:**
- App re-true: Almost never. The single BC is Mach-indexed — it adapts to conditions automatically. Only re-true if you change ammo lots or the barrel has significant round count.

### WG7 Corrections (Capacitor)

No changes needed.

---

## UPDATED CRITICAL RULES FOR BOLT

1. **TM22_TABLE and G1_TABLE must be copied exactly.** Measured data, no rounding.
2. **Scope height in inches.** CZD = 2.6", NOT 1.5".
3. **MV is never a tuning knob.** BC is what the solver adjusts.
4. **Air density: ISA power law** `Math.pow(1 - DA / 145442, 4.256)` — NOT exponential.
5. **Single TM22 BC is the primary prediction engine.** Stepped BC is legacy display only.
6. **Chrono'd MV bypasses temp compensation.** Pass temp=59 to solver when hasChrono is true.
7. **Negative dial-ups rejected.** Error message on single entry, skip silently in collection mode.
8. **HUD state persists across tab switches.** Lifted to App level, saved in localStorage.
9. **localStorage saves on EVERY state change.** No save button.
10. **No confirm() dialogs.** Blocked in WebView. Use inline double-tap.
11. **input type="text" with inputMode="decimal"** for numeric inputs. type="number" is unreliable.
12. **Each truing point stores full conditions snapshot.** Temp, DA, MV, profileMV, tracking correction, zero distance, scope height, BC model, base BC.

---

## SESSION LOG ENTRY

**Session 34 (Feb 17, 2026):**
- Field data from Session 3 (70°F, DA 1007, chrono'd 1068) analyzed
- Bug found: MV double-counting when chrono'd value entered (0.35 → 0.18 MIL avg error)
- Bug found: HUD state lost on tab switch (lifted to App level)
- Bug found: Negative dial-ups accepted (added rejection)
- Bug found: Bolt Guide has wrong air density formula (exponential → ISA power law)
- Single TM22 BC solver built and integrated (replaces stepped BC for predictions)
- Validation: avg ±0.13 MIL, max ±0.23 across 16 points, 3 sessions, 1335ft DA
- L-M Cd knot correction tested — overfits on 9 training points, deferred until more data
- Temp coefficient for SK lot 844 measured at ~0.2 fps/°F (vs 0.8 default)
- Short-range analysis: BC change has <0.01 MIL impact at 100yd, all KZ hits preserved
- Remaining error sources identified: curve shape tilt (40%), MV uncertainty (30%), measurement noise (30%)
- Path to ±0.10: always chrono, 3+ groups averaged, drop ≤125yd, 5+ sessions
