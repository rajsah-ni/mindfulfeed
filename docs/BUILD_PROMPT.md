# Build Prompt — MindfulFeed (Personal Android Reels/Shorts Blocker)

## Role
You are an expert senior Android engineer specializing in Kotlin, Jetpack Compose, and low-level system integration (AccessibilityService, overlays, foreground services). You write clean, idiomatic, production-quality, well-commented code that compiles without manual fixups.

## Context
- **Repository:** `rajsah-ni/mindfulfeed` (default branch `main`).
- **Package / applicationId:** `com.rajsah.mindfulfeed`.
- **Distribution:** Personal use only. Sideloaded debug APK. **NOT for Google Play** — ignore all Play Store policy/approval constraints.
- **A P1–P2 skeleton already exists** in the repo: Gradle setup (Kotlin · Compose · Material 3 · Room · KSP · kotlinx.serialization, min SDK 29 / target 34), `AndroidManifest.xml`, a `FeedDetectionService` (AccessibilityService), `ForegroundMonitor` (UsageStatsManager), a JSON-driven `RuleEngine`, an overlay-based `InterventionManager`, a `BootReceiver`, an onboarding/permissions flow, and `assets/rules.json`.
- The full **SRS is the source of truth** (attach `SRS_MindfulFeed.md`). Where this prompt and the SRS disagree, the SRS wins.

## Mission
Continue from the existing skeleton and implement the app to satisfy the SRS, following the build phases **P3 → P8** in SRS §8. Before writing code, **read the existing repo files** and build on them — do not duplicate or recreate what's already there.

## Hard requirements (do not violate)
1. **No `INTERNET` permission.** The app must compile and run fully offline. No analytics, no crash reporting, no network calls, no accounts. (SRS NFR-1)
2. **Fail-open detection.** If a screen can't be confidently classified as a *blocked* feed surface, do **not** intervene. Never break the user's news feed, DMs, search, or an intentionally opened post/video. Err toward false-negatives. (SRS FR-3)
3. **Per-feed, not per-app.** Blocking toggles target specific feed surfaces (Reels, Shorts, etc.); the rest of each app stays fully usable. (SRS FR-2)
4. **Data-driven rules.** All detection patterns live in editable JSON (`assets/rules.json` + user-editable copy), matching the schema in SRS §6. Target-app UI changes must be fixable by editing rules, not code. Invalid rules are skipped safely, never crash the service. (SRS FR-9)
5. **Non-rooted, stock Android 10+ (API 29+).** No root, no Xposed, no Play Services dependency.
6. **Resilience.** Foreground service restarts after being killed and after reboot. A rule failure for one app must not affect others. (SRS NFR-3)

## What to implement (in order)
- **P3 — Per-feed toggles + allow-list:** Settings-backed enable/disable per feed surface; wire the fail-open `allowIf` logic; add Instagram Reels alongside the existing YouTube Shorts rule. (FR-2, FR-3)
- **P4 — Intervention modes:** `HARD_BLOCK`, `FRICTION` (configurable delay, default 10s; default mode), `AUTO_EXIT` (back/swipe gesture), `GRAYSCALE` overlay. Selectable per feed surface. (FR-4)
- **P5 — Quota & schedule:** Daily feed-time quota with reset time (default midnight); day/time schedule windows; conflicts resolve to the **more restrictive** outcome. (FR-5, FR-6)
- **P6 — Dashboard:** Room-logged events (timestamp, app, surface, intervention, dwell time); Compose dashboard with feed minutes/day, intervention count, clean-day streak, weekly trend chart. All on-device. (FR-7)
- **P7 — Strict mode:** PIN and/or cooldown gate protecting settings, disabling blocks, and stopping the service; optional Device Admin to harden uninstall; Strict-mode settings self-protected. (FR-8)
- **P8 — Rule editor + screen-dump helper:** In-app JSON rule viewer/editor with validation and import/export; a "dump current screen" tool that outputs the active window's node tree (view IDs, text, content-descriptions, class names) to help author/repair rules. (FR-9)
- Polish: Material 3, light/dark theme, clear runtime re-prompt when a required permission is revoked. (FR-11, NFR-4)

## Architecture (keep consistent with the skeleton)
Kotlin · Jetpack Compose (Material 3) · MVVM + unidirectional flow · Coroutines/Flow · Room (events/rules/settings) + DataStore (simple prefs) · AccessibilityService + UsageStatsManager · WindowManager overlay (`TYPE_APPLICATION_OVERLAY`) + `dispatchGesture`/`performGlobalAction` · Foreground Service + `BOOT_COMPLETED` receiver + optional `DeviceAdminReceiver`. Permissions exactly as SRS §5.2 — **and `INTERNET` must be absent.**

## Deliverables
1. All new/updated source files committed, with a logical commit per phase.
2. A `README.md` update: setup, the exact permissions to grant by hand (Accessibility, Usage Access, Display-over-other-apps, Notifications, optional Device Admin), how to build the debug APK, how to edit rules, and how to use the screen-dump tool.
3. The project must **build to a runnable debug APK** with `./gradlew assembleDebug` (add Gradle wrapper files if missing).

## Acceptance criteria (must all pass — from SRS §7)
- **AC-1:** Shorts ON + Friction → opening YouTube Shorts shows a friction overlay within ~0.5s; can't scroll until the delay completes.
- **AC-2:** Shorts ON → a normal YouTube video opened from search plays with **no** intervention.
- **AC-3:** Reels ON → Instagram DMs and home feed are **not** blocked.
- **AC-4:** 10-min quota → exceeding it blocks the feed for the rest of the day; dashboard shows 0 remaining.
- **AC-5:** Strict Mode + PIN → disabling a block requires the PIN (and/or cooldown).
- **AC-6:** After reboot, the detection service is running again with no manual start.
- **AC-7:** Manifest contains **no** `INTERNET`; app works fully offline.
- **AC-8:** When a target-app update breaks a rule, editing the rule JSON in-app restores detection without reinstalling.
- **AC-9:** An unrecognized/uncertain surface is **not** blocked (fail-open).

## Working method
- First, list and read the existing repo files; summarize what's already implemented and your plan, then proceed phase by phase.
- Prefer small, reviewable commits. Keep detection logic data-driven. Comment any view-ID/text patterns as illustrative and discoverable via the screen-dump tool.
- If you must make an assumption, state it briefly in the commit/PR description and choose the option most consistent with the SRS.
