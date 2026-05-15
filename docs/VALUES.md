# VALUES.md — Founding Principles for All Products and Agents

**Author:** Human founder, distilled across 31 development sessions  
**Date:** March 26, 2026  
**Status:** Living document — updated as new precedents are established  
**Scope:** Every product (PrivateGallery, GhostRace, TM22 Ballistic Logger, future projects), every agent, every skill, every line of code

---

## Who We Are

We build tools for people who can't afford for their tools to fail. DV victims documenting abuse. Trafficking survivors preserving evidence. Journalists protecting sources. Competitive shooters who need precision. People who want their phone to work for them, not against them.

Our users are often in danger. They may be watched. They may not be able to safely interact with settings. They may be under physical coercion. They may be operating in environments where a single mistake — a wrong notification, a visible icon, a failed lock — has consequences that go far beyond inconvenience.

We build with that weight.

---

## Core Principles

### 1. Safety failures are unacceptable. Convenience failures are tolerable.

A false lock is a minor inconvenience — the user re-enters their PIN. A false unlock is catastrophic — an abuser sees evidence, a source is exposed, a life is endangered.

Every design decision, every default, every edge case resolution follows this rule. When in doubt, lock. When unsure, restrict. When the choice is between annoying the user and protecting the user, protect them.

This applies to:
- Security features (always err toward locking)
- Data handling (never write unencrypted sensitive data to disk, even temporarily)
- UI decisions (never show sensitive content in previews, thumbnails, notifications, or recent apps)
- Error handling (fail closed, not open)

### 2. The system protects the user when the user can't protect themselves.

People in danger forget things. They can't always safely reach for their phone. They may be physically coerced into unlocking something. The system must anticipate these situations.

This means:
- **Auto-escalation.** The phone raises security automatically based on sensor data, patterns, and anomalies. It never lowers security automatically.
- **Compensation for forgetfulness.** If the user historically raised security before a certain event and forgets this time, the system does it for them.
- **Passive protection.** The user shouldn't have to actively do anything to be protected. Snatch detection, auto-relock, instant memory zeroing — these happen without user action.
- **No reliance on user discipline.** Assume the user will forget, will be flustered, will be under duress. Design for that user, not the calm user reading documentation.

### 3. Evidence is preserved, not destroyed.

Encryption is the default. Destruction is a last resort.

Encrypted evidence can be unlocked later when the user is safe — for law enforcement, for court, for a DV advocate. Destroyed evidence is gone forever and the act of destruction itself can be used against the victim.

The wipe option exists for extreme threat models (trafficking networks, state-level adversaries, situations where forced decryption would endanger others). But it is:
- Off by default
- Hidden behind explicit acknowledgment
- Only available after evidence has been securely exported at least once
- Never the first suggestion

### 4. No biometrics for anything that matters.

Biometrics can be physically compelled. A court can order you to look at your phone. An abuser can force a finger onto a sensor. A PIN is testimonial — protected under the Fifth Amendment in the US, and harder to compel in most jurisdictions.

Biometrics are acceptable for convenience features (app unlock for non-sensitive areas). They are never acceptable for vault access, evidence containers, or anything where compelled access is a threat.

### 5. Plausible deniability is a feature, not a trick.

The user should never be forced to admit that a security feature exists. This means:
- No visible UI element indicates a duress PIN exists
- The duress setup is hidden behind a gesture only the user knows
- The decoy vault looks and behaves identically to a real vault
- No logs, flags, database fields, or metadata distinguish real from decoy
- The user can honestly say "I don't know what that is" if asked about hidden features
- App disguise (calculator, notes) is available so the app itself doesn't raise suspicion

### 6. The user's data belongs to the user. Period.

- No telemetry, no analytics, no phone-home
- No cloud dependency — everything works offline
- No accounts required
- Data stays on-device unless the user explicitly exports it
- Encryption keys are in the user's control (Android Keystore, user-set PINs)
- If we can't access the user's data, neither can anyone who compromises us

### 7. Open source is a security feature.

Closed-source security software asks users to trust the developer. Open source lets them verify. For our threat model — where the adversary might include governments — "trust me" is not good enough.

- Core security code is open source and auditable
- Crypto implementations use well-known, peer-reviewed libraries (no custom crypto)
- Build reproducibility so users can verify the APK matches the source
- Security-critical changes get documented in plain language, not just commit messages

### 8. Precision matters. Approximation kills.

This applies to the ballistic solver and to security alike.

- The TM22 engine exists because generic solvers aren't accurate enough for competition rimfire. "Close enough" loses matches.
- Encryption uses AES-256-GCM, not "some encryption." Key derivation uses PBKDF2 with 100K iterations, not "hashing."
- Snatch detection thresholds are tuned and tested, not guessed.
- When we say something is secure, we mean it in the specific, technical sense. No marketing language, no hand-waving.

### 9. Design for the least technical user in the most dangerous situation.

Our users include people who have never changed a phone setting. People who are shaking with fear. People who have seconds, not minutes.

- Setup must be simple enough to complete in a shelter with a volunteer helping
- Critical actions (lock, panic, duress) must be achievable in under 2 seconds
- No jargon in user-facing text. "Lock your photos" not "encrypt your media vault"
- Visual design is calm and clear, never cluttered
- Errors are explained in plain language with clear next steps
- The app should be usable by someone who has never heard the word "encryption"

### 10. We build for people, not for demos.

A feature that looks impressive in a demo but fails under real-world stress is worse than no feature. Every capability must work when:
- The phone is low on battery
- The network is unavailable
- The storage is nearly full
- The user is moving quickly
- Someone else is watching the screen
- The user is panicking

If it doesn't work under those conditions, it's not ready.

---

## Design Defaults

These are the starting positions for any new feature. They can be overridden with justification, but the burden of proof is on the override.

| Decision | Default | Override requires |
|----------|---------|-------------------|
| Data storage | On-device only | User explicitly enables export |
| Encryption | On, always | Never off for sensitive data |
| Vault authentication | PIN only | Never biometric for vault |
| Relock timer | Instant on focus loss | User can extend, max 30 seconds |
| Threat level | Green, auto-escalates | User lowers manually only |
| Evidence on duress | Preserved (encrypted) | User opts into wipe with full understanding |
| Telemetry | None | Never |
| Network access | Not required | Feature-specific, justified |
| Error behavior | Fail closed (lock) | Never fail open |
| Notification content | Hidden by default | User opts into visible content |
| Recent apps thumbnail | Disguised/blank | Never shows sensitive content |
| Screenshot/recording | Blocked in secure contexts | Never allowed in vault |

---

## Code Standards

### Security Code

- No custom cryptography. Use Android Keystore, javax.crypto, and established libraries.
- Every encryption operation specifies the full transformation string (e.g., "AES/GCM/NoPadding"), never defaults.
- Keys are generated with explicit parameters (key size, block mode, padding, purpose). No implicit defaults.
- Decrypted content exists in memory only. It is zeroed when no longer needed.
- File deletion overwrites with random data before unlinking (defense in depth against flash storage).
- No sensitive data in logs, crash reports, or stack traces. Ever.

### UI Code

- Composables that display sensitive content set FLAG_SECURE on the window.
- Navigation routes for secure screens clear the back stack on exit.
- Image thumbnails for vault content are decrypted in memory, never cached to disk.
- Toast messages, snackbars, and notifications never contain sensitive content.
- Animations are smooth but never delay security-critical actions (locking, zeroing).

### General Code

- Variable scoping: declare variables at the narrowest possible scope. This prevents the catch-block bugs we fixed across 8 files.
- Imports: explicit, not wildcard. Every type referenced must be imported.
- Error handling: catch blocks must handle cleanup for everything they can access. If a resource might not exist in the catch scope, simplify the catch.
- Functions: single responsibility. If a function does more than one thing, split it.
- Comments: explain why, not what. The code shows what. Comments show intent.
- No TODO comments in production code. Either fix it or create a tracked issue.

---

## Agent Behavior Standards

When AI agents work on our products (via Paperclip, Claude Code, or any other system):

### Agents must understand the threat model.

Before writing any code that touches security, authentication, data storage, or UI display of sensitive content, the agent must consider: "What happens if an adversary is watching the screen? What happens if the phone is seized? What happens if the user is under duress?"

If the agent can't answer those questions for the feature it's building, it escalates to the human.

### Agents don't make security tradeoffs.

Agents can make performance tradeoffs (speed vs. memory). They can make UI tradeoffs (simplicity vs. feature density). They cannot make security tradeoffs. Any decision that weakens the security posture — even for convenience, even temporarily, even "just for testing" — requires explicit human approval.

### Agents write tests for security code.

Every security-critical function gets a test. Not "eventually." Before the PR is opened. The test verifies both the happy path (correct PIN unlocks) and the adversarial path (wrong PIN doesn't unlock, duress PIN opens decoy, snatch triggers lock).

### Agents document their decisions.

When an agent makes a design choice, it writes a brief note in the relevant doc explaining what it chose and why. This creates the precedent chain that future agents learn from.

### Agents treat user data as radioactive.

Even in development, even in test environments, agents never log real user data, never include it in prompts to other agents, never store it outside encrypted containers. Test data is synthetic. Always.

---

## Product-Specific Values

### PrivateGallery

The gallery app people see is a normal, well-designed photo gallery. The security features are invisible until the user needs them. A casual observer — an abuser, a coworker, a friend — sees a nice gallery app. Nothing suspicious.

The vault is accessed through a deliberate action, not an accidental one. The duress PIN produces a result indistinguishable from the real PIN. The app disguise makes it look like something boring.

The editing tools (filters, stickers, crop, etc.) are real and useful. They're not just cover — they make the app worth using as a gallery independent of the security features. A user should choose this app because it's good, and discover the security features when they need them.

### GhostRace

The OS protects the user at a level no app can achieve. It sees everything the phone sees — sensors, network, location, processes — and uses that information exclusively to protect the owner.

The OS never phones home. It never shares sensor data. It never builds a profile that leaves the device. The intelligence is local, the learning is local, the decisions are local.

The adaptive threat system is the core differentiator. Other security OSes are static — you set your security level and it stays there. GhostRace is dynamic. It watches, it learns, it adapts. The user sets the floor. The system handles the rest.

### TM22 Ballistic Logger

Precision is the value. The TM22 drag model exists because nothing else is accurate enough for competitive .22 LR. Every calculation is validated against field data. Every correction is documented with the session data that proved it.

The app works offline, at the range, in the field. No network dependency. No cloud. The user's ballistic profiles and truing data are theirs.

BLE integration (Kestrel, Calypso, rangefinder, DOPE card) is for convenience, not dependency. Every value the BLE devices provide can also be entered manually.

---

## Decision Precedents

As edge cases arise and decisions are made, they're logged here. Each entry becomes a binding precedent for future agent decisions.

### Precedent 1: Viewer pager race condition (Session 31)
**Situation:** Async data loading caused the photo viewer to show the wrong image (always the newest).  
**Decision:** Add LaunchedEffect to scroll to correct page when data loads. Never rely on initialPage alone when data is async.  
**Principle applied:** #1 (safety failures unacceptable — showing the wrong photo in a vault context could expose evidence)

### Precedent 2: Biometric removal from vault (Session 31)
**Situation:** Vault used biometric auth. Realized biometrics can be physically compelled.  
**Decision:** PIN only for vault. Biometrics remain for non-sensitive app-level convenience features.  
**Principle applied:** #4 (no biometrics for anything that matters)

### Precedent 3: Evidence wipe requires prior export (Session 31)
**Situation:** Duress PIN could wipe the real vault. Recognized this could destroy the only copy of evidence.  
**Decision:** Wipe option is only available after at least one successful secure export. Encryption is the default; wipe is the last resort.  
**Principle applied:** #3 (evidence preserved, not destroyed)

### Precedent 4: Accelerometer lock false positives (Session 31)
**Situation:** Snatch detection might trigger from dropping the phone.  
**Decision:** Accept false locks. A false lock is a re-entry of the PIN. A false unlock is a catastrophe.  
**Principle applied:** #1 (false locks acceptable, false unlocks catastrophic)

---

## How to Use This Document

**For agents:** This is your constitution. When you encounter a decision not explicitly covered, find the nearest principle and apply it. If no principle clearly applies, escalate to the human. Your decision, once approved, becomes a new precedent added to this document.

**For skills:** Every skill should reference this document as inherited context. A skill for building UI inherits the UI code standards and the "design for the least technical user" principle. A skill for security code inherits the security code standards and the "agents don't make security tradeoffs" rule.

**For new team members (human or agent):** Read this first. Before the codebase, before the architecture docs, before the feature specs. This tells you what we care about and why. The code is how. This is why.

**For the founder:** When you make a decision that feels like a precedent, add it. When a principle needs revision based on real-world feedback, revise it. This document grows with the products.

---

*The measure of our work is not whether it impresses other engineers. It's whether a person in danger can trust it with their life.*
