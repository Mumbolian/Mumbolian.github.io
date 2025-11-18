# iOS App Implementation Plan for T‑Man’s Snooze Schedule

Target: Build a native SwiftUI iOS app that faithfully implements the **production web app at https://t-man.net** (backed by `index.html` + `README.md`, with `UNIFIED_LP_IMPLEMENTATION.md` and `LINEAR_APPROACH.md` as background algorithm notes).

---

## 1. Product Digest (Spec from Production Site)

### 1.1 Core Behavior

The production web app (“T‑Man’s Snooze Schedule”, at `https://t-man.net`) does:

- **Inputs**
  - Wake-up time.
  - Fixed bedtime (non‑negotiable).
  - Preferred number of naps (2 or 3).
  - Mode‑specific nap lengths (distinct configs for 2‑ and 3‑nap days).
  - Wake windows:
    - `lastWake` – fixed last wake window before bed.
    - `minWake`, `maxWake` – biological limits.
    - `maxFirstWake` – upper bound on the first wake window.
  - **Schedule constraints**:
    - “Must be awake by” time.
    - “Must be asleep during” [start, end] window.
  - **Power Sleep Mode**:
    - On/off.
    - Target total daytime sleep (e.g. 180 minutes).
  - Theme toggle (light vs neon).
  - Version (for auto‑update) — currently `APP_VERSION` in `index.html`.

- **Outputs**
  - Up to 3 schedule scenarios:
    - **Build Later** – wake windows increase more aggressively.
    - **Gentle** – modest increases.
    - **Balanced** – roughly equal wake windows.
  - Each scenario:
    - Picks 2 or 3 naps based on preference and feasibility.
    - Distributes wake windows within `[minWake, maxWake]` with a fixed `lastWake`.
    - Ensures:
      - Day starts at wake time.
      - Day ends at (possibly adjusted) bedtime.
    - Optionally:
      - Adjusts nap lengths in 5–30 min increments.
      - Adjusts bedtime by ±15 minutes.
    - Satisfies constraints when feasible (unified LP‑style search).

### 1.2 Invariants from Production Logic

- Wake window parameters (`minWake`, `maxWake`, `lastWake`) are **hard biological constraints** and are never modified to satisfy schedule constraints (see `BUGFIX_WAKE_WINDOW_LIMITS.md`).
- Only nap lengths and bedtime are adjusted for constraint satisfaction (unified approach as implemented in `index.html` and summarized in `README.md` / `UNIFIED_LP_IMPLEMENTATION.md`; `LINEAR_APPROACH.md` is exploratory).
- 2‑/3‑nap mode:
  - Users configure both modes separately.
  - The engine may auto‑switch between them when needed, but user preferences are preserved.
- Power Sleep:
  - Proportionally scales nap lengths so total planned daytime sleep hits a target (without touching wake windows).
- Actual naps:
  - Users can track real nap start/end times.
  - App computes deltas vs planned naps and overall actual vs planned daytime sleep.
  - Future naps are adjusted to “pay back” or “fill in” deficits where possible.

These behaviors define the **acceptance spec** for the iOS app.

---

## 2. Technical Direction for iOS

### 2.1 Stack & Architecture

- **Language**: Swift 5+.
- **UI**: SwiftUI.
- **Minimum iOS**: 16+ (tunable later).
- **Persistence**: `UserDefaults` / `@AppStorage` (mirroring `localStorage` from `index.html`).
- **Layers**:
  - `Domain`: pure scheduling engine and models (no UIKit/SwiftUI).
  - `App`: SwiftUI views, view models, and persistence wiring.

### 2.2 Core Domain Types

- Time unit in engine: **minutes from midnight** (just like the web).
- Key types:
  - `enum NapMode { case two, three }`
  - `struct NapConfig { var nap1, nap2, nap3: Int }`
  - `struct WakeWindowConfig { var last: Int; var min: Int; var max: Int; var maxFirst: Int }`
  - `struct Constraints { var awakeBy: Int?; var asleepStart: Int?; var asleepEnd: Int? }`
  - `struct Settings { ... }` including:
    - `wakeTime`, `bedTime` (as minutes).
    - `preferredMode: NapMode`.
    - `twoNapConfig`, `threeNapConfig`.
    - `wakeWindows: WakeWindowConfig`.
    - `powerSleepEnabled`, `powerSleepTarget`.
    - `theme`.
  - `struct NapScheduleItem { label, start, end, wakeWindow, isAdjusted, isActual, badges }`
  - `struct Scenario { id, name, items: [NapScheduleItem], metadata: ScenarioMetadata }`
  - `struct ActualNap { napNum: Int; start: Int; end: Int }`

---

## 3. Phase 0 – Project Setup & Baseline

### Goals

- Create a SwiftUI iOS project with clear separation between domain and UI.
- Verify basic build/run on simulator and device.

### Tasks

- Create Xcode project, e.g. `NapLogic` (or chosen app name).
- Organize targets:
  - `NapLogic/Domain`
  - `NapLogic/UI`
  - `NapLogic/Support`
- Scaffold:
  - `Settings` model mirroring production schema (based on `README.md` and `index.html`).
  - Empty `ScheduleEngine` type with stubbed methods.
  - `ContentView` showing placeholder text.

### Review Point

- Confirm:
  - App runs on simulator and physical device.
  - Code organization matches the intended Domain vs UI split.

### Acceptance Criteria

- Project builds with no errors.
- App launches to a placeholder home screen.
- Domain folder and types compile cleanly (even if unimplemented).

---

## 4. Phase 1 – Base Scheduling Engine (No Constraints)

### Goals

- Implement a pure Swift engine that reproduces the **base** schedule (no constraints) from the production site at `https://t-man.net`.
- Validate outputs against the live app for core cases.

### Tasks

1. **Time utilities**
   - Implement:
     - `func toMinutes(fromHour: Int, minute: Int) -> Int`
     - `func fromMinutes(_ mins: Int) -> (hour: Int, minute: Int)`
     - `func formatDuration(_ mins: Int) -> String` (e.g. `2h30`).

2. **Wake strategies**
   - Define `enum WakeStrategy { case buildLater, gentle, balanced }`.
   - Approximate the same distribution logic described in `README.md`:
     - Build Later: stronger increase in later wake windows.
     - Gentle: small increases.
     - Balanced: near equal distribution.

3. **Base schedule builder**
   - `struct ScheduleEngine` with:
     - `func buildBaseSchedule(settings: Settings, mode: NapMode, strategy: WakeStrategy) -> ScheduleResult`
   - Behavior:
     - Use user’s nap lengths (unadjusted).
     - Distribute wake windows within `[minWake, maxWake]` and respect `lastWake`.
     - Ensure:
       - Start at `wakeTime`.
       - Final wake window equals `lastWake`.
       - End equals `bedTime`.
     - Populate `NapScheduleItem` list.

4. **Power Sleep (no constraints yet)**
   - Implement `func applyPowerSleep(...) -> [Int]`:
     - Calculate current total nap time.
     - If target > current, increase each nap proportionally (respect minimum nap length).
     - Do not alter wake windows.

5. **Unit tests**
   - Using XCTest:
     - Default 3‑nap settings (as per production defaults: 07:00–20:30, 30/60/30).
     - Default 2‑nap settings (e.g., 90/30).
     - Assert:
       - Sum(wake windows) + sum(naps) = `totalDay - lastWake`.
       - No wake window < `minWake` or > `maxWake`.
       - Final wake window = `lastWake`.

### Review Point

- Compare sample outputs with the live production site at `https://t-man.net` for:
  - Default 3‑nap plan with Power Sleep off.
  - Default 2‑nap plan with Power Sleep off.
- Check visually that nap times and wake windows align (within small acceptable rounding differences).

### Acceptance Criteria

- For default production settings, Swift engine yields schedules that match the live site’s base schedules within ±5 minutes per nap or wake window.
- All engine tests pass and enforce the invariants above.

---

## 5. Phase 2 – SwiftUI Planning UI (No Constraints Yet)

### Goals

- Build a minimal but functional SwiftUI UI for planning a day (matching production inputs/outputs).
- Wire UI to the base `ScheduleEngine`.

### Tasks

1. **SettingsView**
   - Fields:
     - Wake-up time (time picker).
     - Bedtime (time picker).
     - Nap preference (segmented control: 2 vs 3).
     - Nap lengths for current mode (numeric text fields).
     - Wake window settings (collapsible section).
     - Power Sleep toggle and target.
   - Backed by `SettingsViewModel: ObservableObject`.

2. **ScheduleView**
   - Displays:
     - Basic summary line (e.g. total sleep, nap count, earliest/latest nap).
     - List of `NapScheduleItem`s:
       - Label (`Wake-up`, `Nap 1`, `Nap 2`, `Bedtime`).
       - Time range.
       - Badge (wake/nap/bed).
       - Wake window before each segment.

3. **Wiring**
   - `ScheduleViewModel`:
     - Observes `SettingsViewModel`.
     - Calls `ScheduleEngine.buildBaseSchedule(...)` when inputs change.
   - Use `@Published` to trigger SwiftUI updates.

### Review Point

- Manual flow:
  - Change wake-up / bedtime / nap mode / nap lengths.
  - Observe schedule update immediately.
- Confirm:
  - Switching from 2 → 3 naps and back keeps independent nap length settings (like production behavior).

### Acceptance Criteria

- All core inputs map to correct engine parameters.
- Base schedule updates correctly in real time with no crashes.
- Mode-specific nap settings are preserved when toggling between 2 and 3 naps.

---

## 6. Phase 3 – Unified Constraint Solver & Scenario Tabs

### Goals

- Port the unified constraint solving logic from production (`UNIFIED_LP_IMPLEMENTATION.md` / `LINEAR_APPROACH.md`) into Swift.
- Add scenario tabs (“Build Later”, “Gentle”, “Balanced”) that are constraint-aware like the live site.

### Tasks

1. **Constraint model & validation**
   - Implement `Constraints` type.
   - Implement `func validateConstraints(schedule: ScheduleResult, constraints: Constraints) -> Bool`:
     - “Must be awake by”:
       - No nap may overlap or end after the specified awake time.
     - “Must be asleep during [start, end]”:
       - At least one nap must fully cover the interval.

2. **generateNapCombinations in Swift**
   - Mirror the production algorithm:
     - Start with original nap lengths.
     - Pairwise transfers (preserve total nap time).
     - Redistribution (reduce one nap, distribute to others).
     - Single‑nap ± adjustments (affect total nap time).
   - Constrain each nap to min nap length and max adjust (±30).

3. **Unified solver**
   - Implement `func buildConstrainedSchedule(settings: Settings, mode: NapMode, strategy: WakeStrategy, constraints: Constraints) -> ScheduleResult?` that:
     - Iterates candidate bedtime offsets (0, -15, -30, -45, +15, +30, etc.).
     - For each, iterates `generateNapCombinations` results.
     - Calls base `buildBaseSchedule` with candidate nap lengths and bedtime.
     - Returns the first `ScheduleResult` that satisfies constraints via `validateConstraints`.
   - Respect biological wake limits:
     - Never change `minWake`, `maxWake`, or `lastWake`, per `BUGFIX_WAKE_WINDOW_LIMITS.md`.

4. **Multiple scenarios**
   - For each `WakeStrategy`:
     - Compute constrained schedule (or mark that scenario as infeasible).
   - UI:
     - Scenario tabs/segmented control: Build Later / Gentle / Balanced.
     - Only show tabs for scenarios that have a valid schedule.

5. **Constraints UI**
   - `ConstraintsView` with:
     - “Must be awake by” time field.
     - “Must be asleep during” [start, end] time fields.
   - Bind to `Constraints` model in `ScheduleViewModel`.

### Review Point

- Recreate key production test cases and compare against the live site at `https://t-man.net`:
  - “Must be awake by” examples (including cases used when debugging in `IMPLEMENTATION_NOTES.md`).
  - “Must be asleep during 15:00–15:30” example described in the LP docs.
- Validate:
  - Same scenarios are feasible/infeasible.
  - Wake windows never violate `[minWake, maxWake]`.
  - `lastWake` remains fixed.

### Acceptance Criteria

- For each tested constraint scenario where the production site finds a solution, the iOS app finds a solution with:
  - Same nap count.
  - Similar nap/wake structure (within reasonable rounding).
- For impossible cases (according to production logic), the iOS app also reports “no feasible schedule” with a clear message.
- Scenario tabs behave like production: only show feasible scenarios; each scenario may use different adjustments and/or bedtime shifts.

---

## 7. Phase 4 – Auto 2↔3 Nap Switching & Power Sleep Integration

### Goals

- Implement automatic nap count switching logic from production (2 ↔ 3).
- Integrate Power Sleep into constrained schedules.

### Tasks

1. **Mode switching in engine**
   - Extend `ScheduleEngine` or a coordinator:
     - Try preferred `NapMode` with unified constraints.
     - If infeasible:
       - Load stored settings for alternate mode.
       - Optionally apply Power Sleep.
       - Try again with alternate mode.
   - Preserve:
     - `preferredMode` vs `usedMode`.
     - Original nap length preferences for each mode (do not mutate stored settings silently).

2. **Power Sleep integration**
   - Apply Power Sleep before generating nap combinations for constraints.
   - Ensure that all Power Sleep adjustments still respect nap min length and wake window constraints.

3. **UI indicators**
   - Show:
     - A note when the engine auto‑switches nap mode (e.g., “Using 2 naps today to fit your constraints”).
     - A mismatch indicator when preferred mode ≠ used mode.
     - A badge or line summarizing Power Sleep target vs actual total nap time.

### Review Point

- Test flows where the production app at `https://t-man.net` auto‑switches modes:
  - 3‑nap preference that must be downgraded to 2.
  - 2‑nap preference that must be upgraded to 3.
- Confirm Swift outputs match the live site’s behavior for these cases.

### Acceptance Criteria

- In all tested cases:
  - The app uses the same nap count as production for constrained schedules.
  - User’s stored nap configs for both modes remain stable across sessions.
  - Power Sleep produces the same adjusted nap distributions as the live site for given inputs.

---

## 8. Phase 5 – Actual Nap Tracking & Insights

### Goals

- Port actual nap tracking and insights from production (`index.html` logic).
- Allow parents to compare day’s reality vs plan.

### Tasks

1. **Domain logic**
   - Implement:
     - `func adjustedScheduleWithActuals(plan: ScheduleResult, actuals: [ActualNap]) -> ScheduleResult`:
       - Mirror production:
         - Compute net delta between actual and planned for tracked naps.
         - Adjust remaining naps (overshoot vs deficit logic).
       - Preserve wake window limits and `lastWake`.
     - `func computeNapInsights(plan: ScheduleResult, actuals: [ActualNap]) -> NapInsightsResult`:
       - Per‑nap duration, wake window before, delta vs planned.
       - Overall summary (total actual vs planned sleep).

2. **Persistence**
   - Store actual naps in `UserDefaults` under a key analogous to production’s `ACTUAL_NAPS_KEY`.

3. **SwiftUI UI**
   - `ActualNapsView`:
     - Time pickers for Nap 1/2/3 start/end.
     - Insight lines:
       - Duration, wake window, vs planned.
     - Overall summary at bottom.
     - “Clear all” and “Clear this nap” actions.

4. **Integration**
   - Hook `ActualNapsView` into main UI via a button (e.g., “Track today’s naps”).
   - When actuals change:
     - Recompute adjusted plan for remaining naps.
     - Update schedule view.

### Review Point

- Manually replicate a couple of flows from the production site at `https://t-man.net`:
  - Track actual naps that overshoot planned durations.
  - Track naps shorter than planned.
- Compare insights and adjusted future nap lengths/wake windows with the live site.

### Acceptance Criteria

- For the same plan and actuals input, iOS insights match production:
  - Same durations.
  - Same differences vs plan.
  - Same direction of adjustments to future naps.
- No violations of wake window or last wake constraints introduced by actuals logic.

---

## 9. Phase 6 – Theme, Persistence, Copy, and Polish

### Goals

- Implement key UX polish features parallel to production.
- Solidify persistence and app identity.

### Tasks

1. **Theme**
   - Implement a **light theme** echoing production’s watercolor visuals.
   - Optionally, implement a **neon theme** approximation.
   - Persist theme preference via `@AppStorage("theme")`.
   - Apply theme consistently to all views.

2. **Persistence layer**
   - `SettingsStore`:
     - Read/write `Settings` to `UserDefaults`.
     - Schema conceptually mirrors production’s `napPlannerSettings_v6`.
   - `ActualNapsStore`:
     - Read/write `ActualNap` entries.

3. **Copy schedule**
   - Add “Copy schedule” button:
     - Format text similarly to production’s copy output (e.g., “Wake 07:00, Nap 1 09:30–10:00 (30m), …”).
     - Use `UIPasteboard.general` to copy text.

4. **About / Settings screen**
   - Display:
     - App name and version.
     - Link to `https://t-man.net`.
     - Short “How this works” overview.
   - Optionally: minimal FAQ or tips.

### Review Point

- Full apprenticeship run:
  - New user installs app.
  - Configures a typical day, sets constraints, uses Power Sleep, optionally tracks actual naps.
  - Closes app, reopens:
    - Settings, constraints, theme, and recent values persist.

### Acceptance Criteria

- Theme choice persists across launches and applies correctly.
- Settings and actual nap data persist reliably.
- Copy schedule produces a readable, parent‑friendly text version of the day’s plan.

---

## 10. Phase 7 – QA, Performance, and Release

### Goals

- Validate correctness, UX, and performance.
- Prepare the app for TestFlight and App Store release.

### Tasks

1. **Manual QA matrix** (based on production behavior, `README.md` testing notes)
   - Default 3‑nap schedule (no constraints).
   - Toggle 2 ↔ 3 naps; confirm settings preservation.
   - “Must be awake by” scenarios:
     - Simple case where only one nap conflicts.
     - Edge cases where no solution should exist.
   - “Must be asleep during” scenarios:
     - Production’s documented example (“asleep 15:00–15:30”).
   - Power Sleep scenarios:
     - Confirm total planned daytime sleep matches target.
   - Actual naps:
     - Overshoot and deficit cases.
   - Various time formats/locales and time zone behavior.

2. **Performance checks**
   - Confirm schedule recomputation is instant on a mid‑range device.
   - Avoid heavy recomputation on every keystroke where unnecessary (debounce if needed).

3. **Distribution**
   - Apple Developer account setup.
   - App ID, provisioning profiles.
   - App icons & launch screen.
   - App Store Connect metadata:
     - Name, subtitle, description aligned with “planner / optimizer” value prop.
     - Screenshots that show:
       - Planner inputs.
       - Schedule scenarios.
       - Constraints view.
       - Actual nap tracking.

4. **TestFlight**
   - Invite a small group of parents who already use `https://t-man.net`.
   - Ask:
     - “Does this replicate what the site does for you?”
     - “What’s easier or harder vs web?”
     - “Anything missing that you rely on in the browser version?”

### Review Point

- Incorporate TestFlight feedback into a small bug‑fix/polish cycle.
- Re‑run key manual tests after fixes.

### Acceptance Criteria

- No known crash paths or major correctness bugs remain.
- Core features from the production site at `https://t-man.net` (planning, constraints, auto mode switching, Power Sleep, actual nap tracking) are present and behave consistently.
- App passes App Store review and becomes available publicly.

---

## 11. Next Steps

- Choose app name and bundle identifiers (e.g., `NapLogic`, `Nap Blueprint`).
- Start with **Phase 0 & 1**:
  - Get the project compiling.
  - Implement the base schedule engine.
- Once the engine matches `https://t-man.net` for a handful of test cases, move on to constraints and UI.
