<!-- Generated: 2026-07-12 | Updated: 2026-07-12 -->

# TrendyIndicators

## Purpose

Standalone ThinkOrSwim (ThinkScript) indicator repository that ports **capabilities already implemented in PMZBot** onto the TOS chart — so entry timing can be validated visually without the bot running. The study plots the adjusted PMZ band, ±distance admission gate, neutral markers, and CALL/PUT entry triangles at the same bar the bot’s live `PmzEntryStateMachine` would fire.

**Relationship:** this repo is a **TOS presentation surface** for PMZ band + entry timing.

- **Day entries:** methodology parity with PMZBot (`PmzEntryStateMachine`, width clamp, scenarios). The bot only trades **SPX options during the market day**, so it has **no night mode** and is not a source of truth for overnight behavior.
- **Night / after-hours PMZ:** source of truth is the **original Tr3ndy PMZ ThinkScript** in the bot monorepo (band only — no entries), not PMZBot:
  - Canonical: `/opt/dev/peakbot/pmzbot-dev/Documentation/ASSETS/Tr3ndyPMZ.ts.txt`
  - Adjusted variant: `.../Documentation/ASSETS/Tr3ndyPMZ_Adjusted.ts.txt` (same night block + day width clamp)
  - Implemented in `PMZBot_T1T3_EntryIndicator.ts.txt`: `NightMode`, TZ-safe night epochs from cash RTH, form extension, night SM rebind, adjustable clamp.
  - Night *entry* markers are a TOS-only extension of day entry methodology.
- **ES futures session (Globex, ET):** typically **18:00 open → 17:00 next day**, then a ~1h break and reopen at 18:00, repeating **Sun→Mon … Thu→Fri**. Week ends Friday ~17:00; next open Sunday 18:00. **Holidays** can insert an afternoon break (hours not fixed in-repo). Cash RTH and day PMZ/entries sit *inside* that Globex day.

History was filtered from the original `PeakBot-LLC/pmzbot` monorepo path `Documentation/ASSETS/PMZBot_T1T3_EntryIndicator.ts.txt`.

## Bot source of truth (parity)

| Role | Path |
|------|------|
| Canonical bot checkout | **`/opt/dev/peakbot/pmzbot-dev`** |
| Workspace stub often named `pmzbot-dev` | `/opt/dev/pmzbot-dev` — OMC/Claude shell only; **no source tree**. Read/write bot code under `peakbot/pmzbot-dev`. |
| Upstream asset still in monorepo | `/opt/dev/peakbot/pmzbot-dev/Documentation/ASSETS/PMZBot_T1T3_EntryIndicator.ts.txt` (and related distance-chart PNGs) |

### Bot modules this indicator is meant to mirror

| Bot (C#) | Path under `pmzbot-dev` | TOS counterpart |
|----------|-------------------------|-----------------|
| Entry state machine | `Tools/PMZTrader/Strategy/Services/PmzEntryStateMachine.cs` | `script PmzEntryMachine` + study-scope fire flags in `PMZBot_T1T3_EntryIndicator.ts.txt` |
| Width clamp 5–8 pts | `Tools/PMZTrader/Strategy/Calculations/PmzWidthAdjuster.cs` | `AdjustPMZ` / `MinPmzWidthPoints` / `MaxPmzWidthPoints` / `dayAdjustedUpper`/`dayAdjustedLower` |
| Scenario fixtures | `Tools/EntryChartGenerator/TestScenarios.cs` | Manual / analytical parity (no in-repo runner) |
| Unit tests (bot) | `tests/unit/pmztrader/strategy/PmzEntryStateMachine*.cs` | Reference for expected transitions |
| Strategy docs | `Documentation/es-pmz-strategy.md` | Human + agent algorithm reference |
| Chart generator | `Tools/EntryChartGenerator/` | Visual parity for distance-chart style |

### Parity rules for agents

1. **Prefer bot semantics over inventing TOS-only rules.** When behavior is ambiguous, open the C# state machine / width adjuster / `TestScenarios` first.
2. **Scope is entries-only** in TOS: no exits, slot occupancy, cooldown/de-dupe, active-trade caps, consolidation pause (Scenario 13 is bot-side; indicator does not implement it).
3. **v2-only bot features still out of scope** unless explicitly requested: edge-proximity blocking, far-distance 3-minute deferral (Scenario 21), manual PMZ overrides.
4. **Scenarios in indicator header target:** 01–12, 14–20, 22–23 (not 13, not 21 by default).
5. When changing entry logic here, note the matching bot symbol (`IsFullReversal`, pullback budget, pass-through reset, etc.) in the commit/PR so the two codebases stay reviewable as a pair.

## Key Files

| File | Description |
|------|-------------|
| `PMZBot_T1T3_EntryIndicator.ts.txt` | Full ThinkScript study (day + NightMode). Paste into TOS Studies → Create. Day/night PMZ, adjustable clamp, five-state entry machine, distance gate, markers, labels, `Debug*` plots. |
| `PMZBot_T1T3_EntryIndicator_Day.ts.txt` / `_DayOrNight` / `_Auto` | Day-only, mode-flag day/night, and concurrent Auto variants (see README). **Auto WIP:** `DayOnlyMode` + allowlist ES/MES…CL/MCL. |
| `PMZBot_T1T3_EntryIndicator_Auto_RC_DistTpl.ts.txt` | **Preferred Auto paste** — RC night math + alerts + live-entry label + instrument distance template (normal PMZ, family maxD). |
| `PMZBot_T5_ContinuationIndicator.ts.txt` | **Standalone Tier5** study: Trendy Cloud + 1m velocity continuation markers. Not PMZ SM. Stack beside T1–T3; never fold into PMZ fire streams. |
| `Tr3ndyPMZ.ts.txt` | Original Tr3ndy PMZ ThinkScript (day + night/AH band). Canonical night PMZ reference; copied from pmzbot `Documentation/ASSETS/`. |
| `Tr3ndyPMZ_Adjusted.ts.txt` | Same as original plus day `AdjustPMZ` 5–8 clamp; night block unchanged. |
| `README.md` | Operator setup for TOS, out-of-scope list, known limitations, and 2026-07-12 live-testing bug notes (full-reversal arming, aggregation floor, timezone-safe RTH anchors, NaN state-machine self-heal). |
| `AGENTS.md` | This file — AI-oriented map of the repo and agent working rules. |

## Subdirectories

None. Flat repo: PMZ entry studies, **optional T5 continuation study**, Tr3ndy reference scripts, README, AGENTS. Do not invent package/layout structure unless the product is intentionally expanded.

## Architecture (indicator internals)

The single ThinkScript file is the whole product. Logical sections in file order:

1. **Declarations & inputs** — `NightMode`, `futures`, `AdjustPMZ`, `MinPmzWidthPoints`/`MaxPmzWidthPoints`, distance/budget/gate, `NightFormExtensionMinutes`.
2. **Session / timezone anchors** — Cash RTH via `RegularTradingStart`/`End` + `GetTime()`. Night epochs derived (form start, weekday +2h reopen, weekend +50h, long-gap near expected reopen). `Premarket=ALL` day path still platform-clock (known gap).
3. **Day PMZ** — Premarket H/L, gap, RTH lock band, center-preserving clamp.
4. **Night PMZ** — Primary form (Tr3ndy) + extension after reopen when primary invalid/stale; invalid form → no band/entries.
5. **Active band / mode** — NightMode night-only (blank cash RTH); futures band-only when NightMode off; day band in cash RTH when NightMode off.
6. **`script PmzEntryMachine`** — Single SM; session open / fiveSlot / entriesAllowed rebound for day vs night.

### State codes (entry machine)

| Code | Name |
|------|------|
| 0 | `InPMZ_NoPullback` |
| 1 | `InPMZ_CallPullback` |
| 2 | `InPMZ_PutPullback` |
| 3 | `OutsidePmz(Call)` |
| 4 | `OutsidePmz(Put)` |

### Important inputs (production defaults)

| Input | Default | Role |
|-------|---------|------|
| `Premarket` | `"PRE"` | Premarket window for PMZ accumulation (`ALL` is not fully TZ-safe) |
| `NightMode` | `no` | Night-only mode (band + entries); blank in cash RTH |
| `futures` | `no` | When NightMode off: show night band without night entries |
| `AdjustPMZ` | `yes` | Enable width clamp |
| `MinPmzWidthPoints` | `5.0` | Clamp floor (day+night) |
| `MaxPmzWidthPoints` | `8.0` | Clamp ceiling; if Min > Max, clamp skipped |
| `NightFormExtensionMinutes` | `60` | Post-reopen form when primary invalid (Sunday) |
| `MaxDistanceOutsidePmzPoints` | `10.0` | Strict greater-than max distance rejection |

## Night mode requirements (locked 2026-07-12)

Stakeholder Q&A — implement against this; do not re-open without user change.

### Scope / sources of truth

| Topic | Decision |
|-------|----------|
| Bot night trading | **None.** Bot is SPX options, market day only. Not a night parity target. |
| Night **PMZ band** | **Tr3ndy original** math (form/display/gap/20–40%) plus **authorized non-Tr3ndy form extension** when the classic form window has no data (e.g. Sunday open). |
| Night **entries** | Same day methodology (`PmzEntryMachine`, distance, budget, full-reversal). TOS-only. |
| Asset v1 | **Futures-first** (`/ES` Globex). No equity AH preset in v1. |
| Globex context | ~18:00 → 17:00 next day Sun–Fri pattern; holidays not special-cased. |

### Mode matrix

| `NightMode` | `futures` | Behavior |
|-------------|-----------|----------|
| **no** | no | **Day path only** — entry/SM/band behavior bit-identical to current (plus new clamp min/max inputs defaulting 5/8). |
| **no** | **yes** | Day entries as today; **night band can show** (Tr3ndy-style) **without** night entry markers. |
| **yes** | * | **Night only:** night band + night entries; suppress day PMZ band, day markers, day SM. |
| **yes** | no | NightMode alone is enough for night band+entries (futures not required when NightMode on). |

### Session / SM (night)

| Concept | Value |
|---------|--------|
| Form window (primary) | Tr3ndy `fut_time` ≈ **15:55–18:00 ET wall** (cash close tail + maintenance into Globex reopen) |
| Form extension (non-Tr3ndy) | If primary invalid/stale (`now - formStart > 6h`) or weekend reopen: form for `NightFormExtensionMinutes` (0–240, default 60) after reopen; **entries and SM five-min path wait until form frozen** |
| Long-gap reopen | Bar gap ≥12h and first print within 12h after expected weekday/weekend reopen |
| Display window | Primary form and/or post-reopen until next cash open; **cap post-reopen paint at 16h** (no all-weekend hold from Friday) |
| Session open pulse | Primary night: Globex reopen. Extension: freeze/entry-start. Cash open in NightMode clears SM only (blank, no neutral). |
| Entry gate open | Primary: reopen+5m. Extension: max(reopen+5m, extension end). |
| Entry gate close | **Next cash RTH open (~09:30)** — no night entries during cash day |
| NightMode during cash RTH (09:30–15:55) | **Blank** — no day band, no night band hold, no markers. Day stays suppressed while NightMode=yes. Optional banner: `NightMode: outside night window`. |
| Weekend | Extension covers Sunday open; Fri 17:00 week end → no phantom entries without data |
| Shared knobs | Same `MaxDistanceOutsidePmzPoints`, `MaxTriggerCandles`, `EnableFullReversalFreshBreakout`, clamp min/max |
| Clamp | `AdjustPMZ` + `MinPmzWidthPoints`/`MaxPmzWidthPoints` on **day and night** |
| Invalid form | If still no valid ahh/ahl after extension → band NaN, SM idle, no markers (no clamp-of-garbage) |

### Timezone / session rule (locked)

- Human docs may say **ET** (e.g. 18:00 open, 17:00 daily close) because that is how ES Globex is quoted.
- **The study must not depend on the platform’s chosen display timezone.** Bare `SecondsFromTime(1800)` / `SecondsTillTime` against the chart clock face is **invalid** for core day or night gates.
- **ThinkScript has no Globex session API.** Day code already uses `RegularTradingStart`/`End` as **cash RTH** anchors on `/ES` (≈09:30–16:00 absolute), not as Globex 18:00–17:00.
- **Derive night epochs as fixed offsets from cash RTH epochs** (`GetTime()` vs derived absolute times), e.g.:

| Doc label (ET) | Derivation (normal weekday) |
|----------------|----------------------------|
| Cash open 09:30 | `rthStart = RegularTradingStart(date)` |
| Cash close / LIS 16:00 | `rthEnd = RegularTradingEnd(date)` |
| Form start ~15:55 | `rthEnd - 5m` |
| Globex reopen 18:00 | `rthEnd + 2h` |
| Night entry open 18:05 | reopen + 5m |
| Night entry close 09:30 | next `rthStart` |
| Display overnight | from form start / reopen through next `rthStart` |

- Sunday / days without cash RTH on the bar’s date: use **previous session’s `rthEnd`** (or last known cash close) + weekend offset to place reopen — probe on live TOS before merge; do not ship undefined.
- Spec “ET” labels are **documentation only**.

### Post-critic product locks (2026-07-12)

1. **Sunday form:** allow **non-Tr3ndy form extension** (not “empty only”).
2. **Night entries end:** **yes — at next cash open (~09:30)**.
3. **NightMode during cash day:** **blank** (no hold). Primary v1 goal is a clean **night-only** mode via the flag.
4. **Future (not v1):** ideally auto day band+entries in cash RTH and night band+entries overnight **without** a mode flag (or flag becomes optional). That dual calendar is secondary; do not block night-only on it.

### Explicit non-goals (v1)

- Equity session preset, bot overnight execution, dual concurrent day+night entry SMs, holiday afternoon-break calendar.
- Requiring the user to set TOS platform timezone to Eastern for correct night or day gates.
- Pure-Tr3ndy Sunday emptiness (explicitly superseded by form extension).
- Auto day/night switching without `NightMode` (aspirational follow-on; v1 is explicit night-only flag).

### Remaining day inputs (existing study)

| Input | Default | Role |
|-------|---------|------|
| `MaxTriggerCandles` | `5` | 1-minute trigger budget after 5m inside-PMZ pullback |
| `EnableFullReversalFreshBreakout` | `yes` | Outside-to-opposite-outside as clean breakout arm |
| `EntryGateTimeET` | `0935` | Earliest day ET close that may fire entries |
| `Banner` | `no` | Diagnostic `AddLabel` stack (latest bar only) |

## Explicitly out of scope

- Exit markers, consolidation pause, active-trade / cooldown / de-dupe, tier-close simulation (T5 study may apply optional **marker** cooldown only)
- SPX strike/premium selection, IV, DTE (pure ES-price entry timing visual)
- v2-only bot exclusions on **PMZ** path: edge-proximity blocking, far-distance 3-minute deferral, manual PMZ override inputs
- Folding Tier5 into PMZ Auto/Day as a shared fire stream (keep `PMZBot_T5_ContinuationIndicator.ts.txt` standalone)
- Automated unit tests in-repo (parity is against bot `TestScenarios` / live TOS replay, not a local runner)

## For AI Agents

### Working In This Directory

- **Language:** ThinkScript (TOS), not TypeScript/JavaScript. The `.ts.txt` extension is intentional for paste-into-TOS workflows; do not convert to a Node/TS project without an explicit product decision.
- **Edit carefully:** TOS has no local compile/test loop here. Prefer small, comment-backed changes that mirror known C# bot semantics (`PmzEntryStateMachine`, `PmzWidthAdjuster`; T5 → `Tier5MomentumContinuationEvaluator`).
- **PMZ vs T5:** Never merge T5 fires into PMZ `callFire`/`putFire` / live-entry labels. Stack studies on the price subgraph instead.
- **Preserve recursion invariants:** `smState` is `CompoundValue(1, PmzEntryMachine(smState[1], ...), 0)`. Any `NaN` on a mid-chart bar poisons all subsequent bars unless `statePrev`/`armPrev`/`budgetPrev` self-heal to `0` — do not remove those guards.
- **Full-reversal / cross-through:** Must key off **raw previous state** (`statePrev` from `smState[1]`), not a zeroed `stateForFiveMinute`. Zeroing previous state on cross-through bars reintroduces a silent one-minute entry delay (fixed 2026-07-12).
- **Aggregation:** Floor multi-period series to the chart’s primary period when needed so non-1m charts show the warning banner instead of `"Secondary period cannot be less than primary"`.
- **Session times:** Prefer epoch anchors (`RegularTradingStart`/`End`, `GetTime`) for any new RTH/premarket/entry-gate logic. Do not reintroduce bare `SecondsFromTime(0930)`-style comparisons for core gates.
- **Parity target:** bot `TestScenarios` 01–12, 14–20, 22–23 (full-reversal/cross-through: 07, 08, 09, 23 especially). Source: `/opt/dev/peakbot/pmzbot-dev/Tools/EntryChartGenerator/TestScenarios.cs`. There is no `PullbackConfirmation` input — always strict pullback mode.
- **Do not treat `/opt/dev/pmzbot-dev` as the codebase** — that path is a workspace stub; open `/opt/dev/peakbot/pmzbot-dev` for bot sources.
- **Copyright header:** File retains Tr3ndy/Simpler Trading provenance plus Peakbot adjustment notes; keep the header block intact when editing.
- **Whitespace/commits:** If this environment applies `dotnet format` rules elsewhere, they do not apply to ThinkScript; do not run C# formatters on this file.

### Testing Requirements

There is no automated test harness in this repo. After substantive logic changes, agents should:

1. **Static parity check** — Trace state transitions against the bot’s C# `PmzEntryStateMachine` / width adjuster rules and known scenario IDs.
2. **TOS import smoke** — User pastes full file into Studies → Create on **continuous `/ES`**, **1-minute**, extended hours, multi-day lookback (include prior Friday for Sunday night); study on **price subgraph**.
3. **Diagnostics** — `Banner` for current bar; Data Window `DebugState`, `DebugCleanArm`, `DebugBudget`, `DebugGateOpen`, `DebugOneAbove`/`Below`, `DebugCallDist`/`PutDist`, `DebugNightMode`, `DebugNightFormValid`/`Frozen`, `DebugPrimaryLocked`, `DebugInExtension`, `DebugInNightDisplay`, `DebugFiveSlot`, `DebugNightEntries`, `DebugReopenKind` (1=weekday, 2=weekend, 3=long-gap).
4. **Regression watchlist** — Day full-reversal; NightMode blank cash RTH; Sunday extension freeze before entries; Central-time alignment; NaN self-heal; `AdjustPMZ=no` ordered band.

### Common Patterns

- **Packed recursive state:** `state + 10*cleanArm + 100*triggerBudget` in one `def`, stepped via `script` + `CompoundValue`.
- **Hidden scale anchor:** `plot ScaleAnchor = close; ScaleAnchor.Hide();` keeps overlays on the price axis.
- **Display flags:** Many plots are gated by `Show*` inputs and/or `ShowDistanceChartStyle`.
- **Strict distance:** CALL distance = `close - PMZ Low`; PUT distance = `PMZ High - close`; reject when **strictly greater than** `MaxDistanceOutsidePmzPoints`.
- **Day entry gate:** `nowEpoch >= dayEntryGateEpoch and nowEpoch < premarketResetEpoch`.
- **Night entry gate:** `nightFormFrozen` and `nightEntryStartEpoch` → `nightEntryCloseEpoch` (extension freezes SM until form ends).
- **Crossed-boundary resets:** Prefer `nowEpoch >= X and nowEpoch[1] < X` over `SecondsTillTime(X) == 0` (fragile on missing bars).

## Dependencies

### Internal

- **Behavioral dependency (read-only):** PMZBot at `/opt/dev/peakbot/pmzbot-dev` — especially `PmzEntryStateMachine`, `PmzWidthAdjuster`, and `Tools/EntryChartGenerator/TestScenarios.cs`.
- No other source modules in this repository; do not vendor the bot into this tree.

### External

- **ThinkOrSwim / thinkorswim ThinkScript** runtime (TD Ameritrade / Schwab platform) — only execution environment.
- Chart symbol: continuous **`/ES`** futures; aggregation: **1 minute**; extended/premarket hours required for PMZ seed.
- Upstream visual lineage: Tr3ndy PMZ (trendytrading.co / Simpler Trading) with Peakbot risk-adjustment and entry-machine extensions.

## Known open items

- Day `Premarket="ALL"` still uses platform-clock literals (PRE path is epoch-safe).
- Holiday Globex afternoon breaks not calendared; Friday cashClose+2h rarely has bars (Sunday uses Fri-midnight+2d+18h / long-gap). Absolute-ms day math can still skew ~1h around DST transitions.
- Full TOS replay of TestScenarios / live bot-log spot-check still outstanding for day path.
- Auto concurrent day+night is implemented in `_Auto` (no `NightMode`). Optional `DayOnlyMode=yes` forces day-only on any symbol. Night allowlist: ES/MES, NQ/MNQ, RTY/M2K/TF, YM/MYM, **CL/MCL** (customer/Tr3ndy PMZ on oil); other futures products are day-only by design (see README).
- No in-repo automated ThinkScript runner.

<!-- MANUAL: Any manually added notes below this line are preserved on regeneration -->
