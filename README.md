# MatchOS Monolith

The current, canonical source for the **MatchOS** ballistic logger app — the version installed on Don's phone today. Single-file Capacitor web app (React from CDN, BLE plugin) targeting Android.

**Repo status:** maintenance only. Active development has moved to [MatchOS-Refactor](https://github.com/Abraxas2506/MatchOS-Refactor) (native Kotlin/Compose rebuild). This repo exists so we have one stable place to read the working app's source while the rebuild catches up.

## What's here

```
app/tm22-app/
  www/index.html        — the entire app (~2.4k lines)
  capacitor.config.json — appId com.tm22.ballistics, appName MatchOS
  package.json          — Capacitor 6.x + @capacitor-community/bluetooth-le
  BUILD_GUIDE.md        — how to npx cap sync + open in Android Studio

docs/
  HANDOFF-22LR-BALLISTIC-CALC.md  — comprehensive math + design handoff (54 KB)
  TM22_BUILD_GUIDE.md             — original Capacitor build recipe
  BOLT-BUILD-GUIDE.md             — bolt.new scaffold history
  SESSION-34-FINDINGS.md          — feb 16 session notes
  SESSION-35-FINDINGS.md          — feb 17–18 field data S4 + wind log + dev tab
  VALUES.md                       — application values / product principles
```

## App architecture (one-liner)

`<title>MatchOS</title>` → `App` component → 6 tabs: **HUD · Truing · Profiles · BLE · Guide · Dev**.

BLE devices supported: TM22 DOPE Card, Calypso Wind, Kestrel 5700, Sig Kilo 3K (rangefinder).

Solver: TM22 Cd(Mach) drag table + single-BC tuning via truing data, spin drift add-on, atmospheric compensation via Kestrel or manual.

## Known issues (in this revision)

- **Wind hold "never dies"** — wind hold display may not return to zero even with zero wind input. Suspect: `totalWindMils = wDisp + Math.abs(spinMils)` at `app/tm22-app/www/index.html:1449` — spin drift is unconditionally added with `Math.abs`, so the displayed combined hold can never read 0. Tracked separately.

## Relationship to other repos

| Repo | Role |
|---|---|
| **MatchOS-Monolith** (this) | The shipping/canonical monolith. The thing on Don's phone. |
| [MatchOS-Refactor](https://github.com/Abraxas2506/MatchOS-Refactor) | Native Kotlin/Compose rebuild — the eventual replacement. |
| [Ballistic-Project](https://github.com/Abraxas2506/Ballistic-Project) | Older fork (V3 APK + AdaptiveBallisticLogger JSX). Archive. |
| SDL-MatchOS (legacy) | Earlier TypeScript modular rewrite — architectural blueprint. |

When MatchOS-Refactor ships at parity, this repo and the others are retired and the Refactor repo becomes the single source of truth.
