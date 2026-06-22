# Software Requirements Specification (SRS)
## Project: "MindfulFeed" — Personal Android Reels/Shorts Blocker

**Version:** 1.0  
**Author:** rajsah-ni  
**Date:** 2026-06-22  
**Status:** For Implementation  
**Distribution:** Personal use only — sideloaded APK. **Not for Google Play.**

---

## 1. Introduction

### 1.1 Purpose
MindfulFeed is a personal Android application that **surgically removes short-form, infinite-scroll video feeds** (Instagram Reels, YouTube Shorts, Facebook Reels, Snapchat Spotlight, Reddit video feed) from existing social apps, **while preserving access to informational content** — news feeds, direct messages, search, posting, and individual content the user intentionally opens.

The goal is **not** to block whole apps. It is to eliminate the addictive, dopamine-driven scroll surfaces while keeping the apps usable as sources of news and communication.

### 1.2 Scope
The app runs locally on a single user's Android device. It uses Android's `AccessibilityService`, `UsageStatsManager`, and overlay (`SYSTEM_ALERT_WINDOW`) capabilities to detect and intervene on short-form feed surfaces in real time.

**In scope:**
- Real-time detection of short-form feed surfaces in target apps.
- Configurable, per-feed blocking (not per-app).
- An explicit "keep my news/info feed" allow philosophy.
- Multiple intervention styles (hard block, friction prompt, auto-exit, grayscale).
- Local usage tracking and a habit dashboard.
- Scheduling and daily quotas.
- Anti-impulse / strict mode.
- 100% on-device operation; no network, no accounts, no analytics.

**Out of scope:**
- Google Play distribution and associated policy compliance.
- Cloud sync, multi-device support, user accounts.
- Modifying or injecting into the target apps' code (no root, no Xposed required; must work on stock, non-rooted Android).
- Blocking web browsers or desktop.

### 1.3 Definitions
| Term | Meaning |
|---|---|
| **Target App** | A social app MindfulFeed monitors (Instagram, YouTube, etc.). |
| **Feed Surface** | A specific addictive scroll screen inside a target app (e.g., the Reels tab). |
| **Allowed Surface** | An informational screen to be left untouched (home/news feed, DMs, search, a single opened post/video). |
| **Intervention** | The action taken when a blocked Feed Surface is detected. |
| **Rule** | A pattern (view ID, text, content-description, class name) used to identify a Feed Surface. |
| **Strict Mode** | A state where settings cannot be changed/disabled on impulse. |

### 1.4 Assumptions & Constraints
- **C1:** Device runs **Android 10 (API 29) or newer**, non-rooted.
- **C2:** User will manually grant Accessibility, Usage Access, "Display over other apps," and notification permissions.
- **C3:** Target apps change their UI frequently; detection **Rules must be editable by the user without rebuilding the app** (see FR-9).
- **C4:** App is installed via sideloaded APK; no Play Services dependency is required.
- **A1:** Only one user; no authentication beyond the optional Strict Mode lock.

---

## 2. Overall Description

### 2.1 Product Perspective
A standalone, native Android app composed of:
1. A **detection engine** (Accessibility + Usage Stats) running as a persistent foreground service.
2. An **intervention layer** (overlay window + simulated gestures).
3. A **local data store** (Room DB) for settings, rules, and usage logs.
4. A **configuration UI** (Jetpack Compose) for the user.

### 2.2 High-Level Flow
```
UsageStatsManager / foreground event
        │  (a target app comes to foreground)
        ▼
AccessibilityService activates inspection
        │  (parse active window's view hierarchy)
        ▼
Match against Rules for blocked Feed Surfaces
        │
   ┌────┴─────┐
 No match   Match found
   │            │
 Allow      Apply Intervention (per user config):
 (do          • Hard block overlay
 nothing)     • Friction / confirm prompt
              • Auto-exit (back/swipe gesture)
              • Grayscale filter
        │
        ▼
Log event to local DB → Dashboard
```

### 2.3 User Characteristics
Single technical user (the author) who can grant special permissions and edit detection rules if needed.

---

## 3. Functional Requirements

### FR-1 — Detection Engine (Core)
- **FR-1.1** The app SHALL run a persistent foreground service that survives reboots (re-arm via `BOOT_COMPLETED`).
- **FR-1.2** The app SHALL use `UsageStatsManager` (or accessibility window-state events) to detect when a Target App enters the foreground, to avoid running heavy inspection continuously.
- **FR-1.3** When a Target App is foregrounded, the app SHALL use `AccessibilityService` to read the active window's node tree and evaluate it against the active Rule set.
- **FR-1.4** Detection latency SHALL be ≤ 500 ms from the Feed Surface becoming visible to the Intervention firing, under normal conditions.
- **FR-1.5** The engine SHALL re-evaluate on relevant accessibility events (`TYPE_WINDOW_STATE_CHANGED`, `TYPE_WINDOW_CONTENT_CHANGED`) and debounce to avoid excessive CPU use.

### FR-2 — Per-Feed Blocking (Not Per-App)
- **FR-2.1** The user SHALL be able to independently enable/disable blocking for each Feed Surface:
  - Instagram → Reels
  - YouTube → Shorts
  - Facebook → Reels / Video tab
  - Snapchat → Spotlight
  - Reddit → video/short feed
  - (Architecture SHALL allow adding more targets via config.)
- **FR-2.2** Disabling blocking for an app's Feed Surface SHALL have **zero effect** on the rest of that app.

### FR-3 — Allowed-Surface Preservation (Critical)
- **FR-3.1** The app SHALL explicitly treat the following as **Allowed Surfaces** and never intervene on them: home/news feed, stories, direct messages/chat, search/explore (non-reel results), a single intentionally-opened post or video, profile, notifications, settings.
- **FR-3.2** **Fail-open principle:** if the engine cannot confidently classify a surface as a *blocked* Feed Surface, it SHALL default to **allowing** it, to avoid breaking the user's news/information access. Detection must err toward false-negatives, not false-positives.

### FR-4 — Intervention Modes
The user SHALL select, per Feed Surface, one of the following Intervention modes:
- **FR-4.1 Hard Block:** Render a full-screen (or feed-region) opaque overlay via `SYSTEM_ALERT_WINDOW` with a custom message and a "Go back" / "Close app" button.
- **FR-4.2 Friction Prompt:** Show a dismissible overlay requiring a deliberate action (e.g., wait N seconds, then tap "I still want to continue") before the feed is accessible. Configurable delay (default 10s).
- **FR-4.3 Auto-Exit:** Dispatch a gesture (`GLOBAL_ACTION_BACK` or a programmatic swipe) to leave the Feed Surface automatically.
- **FR-4.4 Grayscale:** Apply a grayscale/desaturation overlay over the feed region to make it unappealing without fully blocking it.
- **FR-4.5** Default mode SHALL be **Friction Prompt**.

### FR-5 — Daily Quota / Mindful Breaks
- **FR-5.1** The user SHALL optionally allow a configurable amount of feed time per day (e.g., 10 minutes), tracked cumulatively, after which the chosen Intervention activates for the rest of the day.
- **FR-5.2** Quota SHALL reset at a user-defined time (default local midnight).
- **FR-5.3** Remaining quota SHALL be visible in the UI and optionally in the intervention prompt.

### FR-6 — Scheduling
- **FR-6.1** The user SHALL define time windows / day-of-week schedules during which blocking is active (e.g., work hours, bedtime) and windows where it is relaxed.
- **FR-6.2** Conflicting schedule + quota rules SHALL resolve to the **more restrictive** outcome.

### FR-7 — Usage Tracking & Dashboard
- **FR-7.1** The app SHALL log, locally, each detection event: timestamp, target app, feed surface, intervention taken, and dwell time before intervention.
- **FR-7.2** The dashboard SHALL display: feed minutes per day (and reels-vs-allowed split where measurable), number of interventions, "clean streak" (consecutive days under quota), and weekly trend chart.
- **FR-7.3** All stats SHALL be computed and stored entirely on-device.

### FR-8 — Strict / Anti-Impulse Mode
- **FR-8.1** When Strict Mode is ON, changing settings, disabling a block, or stopping the service SHALL require passing a friction gate: a user-set PIN, a cooldown timer (e.g., 5 min), or both.
- **FR-8.2** The app SHALL make casual uninstallation harder by optionally enabling Device Admin (so it must be deactivated before uninstall) — clearly documented as personal-use, reversible.
- **FR-8.3** Strict Mode settings themselves SHALL be protected by the same gate.

### FR-9 — Editable Detection Rules (Resilience)
- **FR-9.1** Detection Rules (view IDs, text patterns, content-descriptions, class names, package names) SHALL be stored in a human-readable config (JSON) that the user can view and edit **without recompiling**.
- **FR-9.2** The app SHALL provide an in-app editor (or an import/export to a file) for Rules, with validation and a "test against current screen" helper that dumps the current window's node tree to help author new Rules.
- **FR-9.3** Invalid Rules SHALL be skipped safely and never crash the service.

### FR-10 — Custom Messaging
- **FR-10.1** The user SHALL be able to set custom text shown in Hard Block and Friction interventions (e.g., a personal reminder of why they're cutting back).

### FR-11 — Onboarding & Permissions
- **FR-11.1** A first-run flow SHALL guide the user to grant: Accessibility Service, Usage Access, Display-over-other-apps, Notifications, and (optionally) Device Admin — each with a clear explanation and a deep link to the relevant settings screen.
- **FR-11.2** The app SHALL detect at runtime if any required permission is revoked and prompt to re-grant, while degrading gracefully.

---

## 4. Non-Functional Requirements

### NFR-1 — Privacy (Hard Requirement)
- **NFR-1.1** The app SHALL request **no `INTERNET` permission**. It SHALL function fully offline.
- **NFR-1.2** No data (usage, screen content, rules) SHALL ever leave the device. No analytics, no crash reporting to third parties, no accounts.
- **NFR-1.3** Screen content read via Accessibility SHALL be used only transiently for classification and SHALL NOT be persisted beyond the minimal event log in FR-7.1 (which stores metadata, not screen content).

### NFR-2 — Performance & Battery
- **NFR-2.1** Idle CPU usage SHALL be negligible when no Target App is in the foreground.
- **NFR-2.2** The foreground service SHALL be optimized (event-driven + debounced) to avoid measurable battery drain in everyday use.

### NFR-3 — Reliability
- **NFR-3.1** The service SHALL automatically restart after being killed by the system and after reboot.
- **NFR-3.2** A failure in Rule evaluation for one app SHALL NOT affect monitoring of other apps.

### NFR-4 — Usability
- **NFR-4.1** The configuration UI SHALL be usable in under 5 minutes for the core task (turn on Reels + Shorts blocking with Friction mode).
- **NFR-4.2** Material 3 design, light/dark theme support.

### NFR-5 — Maintainability
- **NFR-5.1** Target-app detection logic SHALL be data-driven (FR-9) so UI changes in target apps are fixed by editing config, not code.
- **NFR-5.2** Adding a new Target App SHALL require only a new Rule entry + (optionally) a new intervention preset.

### NFR-6 — Compatibility
- **NFR-6.1** Support Android 10 (API 29) → latest, on non-rooted stock devices.
- **NFR-6.2** No dependency on Google Play Services.

---

## 5. Technical Architecture (Recommended)

| Layer | Technology |
|---|---|
| Language | **Kotlin** |
| UI | **Jetpack Compose**, Material 3 |
| Architecture | MVVM + unidirectional data flow |
| Detection | `AccessibilityService` + `UsageStatsManager` |
| Intervention | `WindowManager` overlay (`TYPE_APPLICATION_OVERLAY`), `dispatchGesture`, `performGlobalAction` |
| Persistence | **Room** (settings, rules, usage events) + DataStore (simple prefs) |
| Background | Foreground `Service`, `BOOT_COMPLETED` receiver, optional `DeviceAdminReceiver` |
| Async | Kotlin Coroutines + Flow |
| Min SDK | 29 · Target SDK | latest stable |
| Distribution | Debug/release **APK**, sideloaded |

### 5.1 Key Components
- `FeedDetectionService : AccessibilityService` — core engine (FR-1, FR-3).
- `ForegroundMonitor` — wraps `UsageStatsManager` (FR-1.2).
- `RuleEngine` — loads JSON rules, matches node trees (FR-9).
- `InterventionManager` — overlays, gestures, grayscale (FR-4).
- `QuotaScheduler` — quota + schedule resolution (FR-5, FR-6).
- `UsageRepository` (Room) — event logging + dashboard data (FR-7).
- `StrictModeGuard` — PIN/cooldown gate + Device Admin (FR-8).
- Compose screens: Onboarding, Home/Dashboard, Per-Feed Settings, Rule Editor, Strict Mode, Schedule/Quota.

### 5.2 Required Permissions
- `BIND_ACCESSIBILITY_SERVICE` (service declaration)
- `PACKAGE_USAGE_STATS` (Usage Access — user-granted)
- `SYSTEM_ALERT_WINDOW` (overlays)
- `FOREGROUND_SERVICE` (+ specific subtype as required by target SDK)
- `RECEIVE_BOOT_COMPLETED`
- `POST_NOTIFICATIONS`
- (optional) Device Admin via `DeviceAdminReceiver`
- **Explicitly NOT** `INTERNET`.

---

## 6. Example Rule Schema (for FR-9)

```json
{
  "rules": [
    {
      "id": "youtube_shorts",
      "enabled": true,
      "package": "com.google.android.youtube",
      "label": "YouTube Shorts",
      "match": {
        "anyOf": [
          { "viewIdContains": "reel_recycler" },
          { "viewIdContains": "shorts" },
          { "textEquals": "Shorts" },
          { "contentDescContains": "Shorts" }
        ]
      },
      "allowIf": [
        { "viewIdContains": "search" },
        { "viewIdContains": "comments" }
      ],
      "intervention": "FRICTION"
    },
    {
      "id": "instagram_reels",
      "enabled": true,
      "package": "com.instagram.android",
      "label": "Instagram Reels",
      "match": {
        "anyOf": [
          { "viewIdContains": "clips_viewer" },
          { "contentDescContains": "Reels" },
          { "textEquals": "Reels" }
        ]
      },
      "allowIf": [
        { "viewIdContains": "direct" },
        { "viewIdContains": "feed_timeline" }
      ],
      "intervention": "HARD_BLOCK"
    }
  ]
}
```
> Note: `viewId`/text values above are illustrative. Final values must be discovered via the FR-9.2 "dump current screen" helper, since target apps change them over time.

---

## 7. Acceptance Criteria

| # | Given / When / Then |
|---|---|
| AC-1 | Given Shorts blocking is ON in Friction mode, when I open YouTube Shorts, then within ~0.5s a friction overlay appears and I cannot scroll until I complete the delay. |
| AC-2 | Given Shorts blocking is ON, when I open a normal YouTube video from search, then no intervention fires and playback is normal. |
| AC-3 | Given Reels blocking is ON, when I open Instagram DMs and the home feed, then nothing is blocked. |
| AC-4 | Given a daily quota of 10 min, when I exceed it, then the feed is blocked for the rest of the day and the dashboard shows 0 remaining. |
| AC-5 | Given Strict Mode ON with a PIN, when I try to disable Reels blocking, then I must enter the PIN (and/or wait the cooldown) first. |
| AC-6 | Given the device reboots, when it finishes booting, then the detection service is running again without manual start. |
| AC-7 | Given the app is installed, when I inspect its manifest/permissions, then `INTERNET` is absent and the app works fully offline. |
| AC-8 | Given a target app updates and a Rule stops matching, when I edit the Rule JSON in-app, then detection resumes without reinstalling the app. |
| AC-9 | Given an unrecognized surface, when its classification is uncertain, then the app does NOT block it (fail-open). |

---

## 8. Build Phases (suggested order)

1. **P1 — Skeleton:** Project setup, foreground service, permissions onboarding, `UsageStatsManager` foreground detection.
2. **P2 — First detection:** AccessibilityService + RuleEngine detecting **YouTube Shorts only**, with Hard Block overlay. Validate AC-1, AC-2.
3. **P3 — Per-feed + allow-list:** Add per-feed toggles and Allowed-Surface fail-open logic (FR-2, FR-3). Add Instagram Reels.
4. **P4 — Interventions:** Friction, Auto-Exit, Grayscale modes (FR-4).
5. **P5 — Quota & schedule:** FR-5, FR-6.
6. **P6 — Dashboard:** Room logging + Compose charts (FR-7).
7. **P7 — Strict Mode:** PIN/cooldown + Device Admin (FR-8).
8. **P8 — Rule editor + screen-dump helper** (FR-9), polish, dark theme.

---

## 9. Risks & Mitigations
| Risk | Mitigation |
|---|---|
| Target apps change view IDs → detection breaks | Data-driven Rules (FR-9) + in-app screen-dump tool to re-author quickly. |
| Over-blocking breaks news access | Fail-open principle (FR-3.2); allow-lists per rule. |
| System kills foreground service | Foreground notification + boot receiver + restart logic (NFR-3). |
| Accessibility misuse concerns | Personal device only; no data leaves device (NFR-1). |
| Self-bypass on impulse | Strict Mode + Device Admin (FR-8). |