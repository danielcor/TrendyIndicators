# PMZBot / TrendyIndicators — TOS PMZ entry studies

ThinkOrSwim ThinkScript studies that plot PMZ bands and CALL/PUT entry markers for the
PMZBot-style entry state machine (entries only; bot does not need to be running).

**Reference (night band math):** `Tr3ndyPMZ.ts.txt` / `Tr3ndyPMZ_Adjusted.ts.txt`

## Study files

| File | Role |
|------|------|
| `PMZBot_T1T3_EntryIndicator.ts.txt` | **Day only** (canonical name; same as `_Day`) |
| `PMZBot_T1T3_EntryIndicator_Day.ts.txt` | **Day only** — cash RTH entries, bot day parity |
| `PMZBot_T1T3_EntryIndicator_DayOrNight.ts.txt` | **Day *or* night** via `NightMode` / `futures` flags (not both at once) |

## Day only

- Premarket PMZ (≈07:25–09:30), 5–8 pt width clamp (hardcoded in day study), entry gate 09:35.
- Five-state pullback admission (bot `PmzEntryStateMachine` methodology).
- Cash RTH epoch anchors (`RegularTradingStart`/`End`) for timezone-safe day gates.

## Day-or-night (`_DayOrNight`)

### Day mode (`NightMode = no`, default)

- Same day entry methodology as day-only, plus **adjustable** clamp (`MinPmzWidthPoints` /
  `MaxPmzWidthPoints`, defaults 5–8).
- Optional `futures = yes`: show the **night PMZ band only** (no night entry markers).

### Night mode (`NightMode = yes`)

- **Night-only:** suppresses day band and day entries. During cash RTH the study is **blank**.
- Night PMZ form follows Tr3ndy with **Sunday form extension** (`NightFormExtensionMinutes`).
- Night entries use the same state machine; clocks derived from cash RTH epochs (not platform TZ).
- Shared knobs: distance, trigger budget, full-reversal, width clamp min/max.

## Explicitly out of scope

- Exit markers, consolidation pause, active-trade/cooldown/de-dupe awareness, tier-close
  simulation — entries only.
- SPX strike/premium selection, IV, DTE — this is a pure price entry-timing visual, not a
  trade sizing tool.
- Bot overnight execution (bot is SPX day-only).

## Setup in TOS

1. Chart: **TOS continuous `/ES`** (or equity with extended hours for day), **1-minute**,
   extended/pre-market hours enabled, multi-day lookback.
2. Studies → Edit Studies → Create → paste the chosen study file → Apply.
3. Confirm the study is on the **price subgraph** (not a separate lower panel) — if PMZ/gate
   lines don't track the ES price axis, use the study's gear icon → Move to → Existing
   subgraph → Price.
4. Turn on `Banner` (input, off by default) for the full diagnostic label stack (`State`,
   `CleanArm`, `Budget`, `Gate`, `LIS`, `PMH/PML Time`, etc). Note: `AddLabel` only ever shows
   the **latest/current bar's** values, not whatever bar you've scrolled to — it does not
   follow the chart viewport or cursor.
5. To inspect a **historical** bar's internal state, hover the crosshair over that bar and
   check the Data Window/tooltip for the hidden `Debug*` plots (`DebugState`, `DebugCleanArm`,
   `DebugBudget`, `DebugGateOpen`, `DebugOneAbove`/`Below`, `DebugCallDist`/`PutDist`,
   `DebugNightMode`, `DebugNightFormValid`/`Frozen`, `DebugPrimaryLocked`, `DebugInExtension`,
   `DebugInNightDisplay`, `DebugFiveSlot`, `DebugNightEntries`, `DebugReopenKind` 1=weekday /
   2=weekend / 3=long-gap) — unlike labels, plots are inspectable on any bar. Enable plot names
   via the study gear/eye if TOS hides them.
6. **Lookback:** for Sunday night, load chart history through the prior **Friday 15:55+** so
   `lastRthEnd` can latch (cash-close anchor).

## Known limitations / open items

- **Timezone:** day and night **core** gates use cash RTH epoch anchors + fixed offsets
  (independent of platform display timezone). Non-default day `Premarket = ALL` still uses
  platform-clock literals for day accumulation only.
- **ThinkScript has no Globex API:** night reopen is `cashClose+2h` (Mon–Thu), `+50h` (Fri→Sun
  style), and/or a long bar-gap reopen detector. Holiday afternoon breaks are not calendared.
- **Validation:** parity against the bot's 23 `TestScenarios.cs` fixtures has only been
  checked analytically (code trace) for the full-reversal/cross-through scenarios (07, 08, 09,
  23) — a real TOS replay of the full scenario set, a side-by-side PNG comparison, and a live
  session spot-check against bot log timestamps are all still outstanding.
- Always **strict pullback** mode (no `PullbackConfirmation` input).

## Debugging notes from initial live TOS testing (2026-07-12)

The original merged version (PR #217, `pmzbot` repo) compiled but had never actually been
validated against live TOS market data. A single testing session surfaced four real,
independent bugs, in the order found:

1. **Full-reversal / cross-through arming was self-defeating.** `stateForFiveMinute` was
   forced to `0` on any 5-minute candle that crossed through the PMZ band and closed outside
   it — which is exactly the condition `fullReversalToCall/Put` and `crossThroughCleanCall/Put`
   needed the *real* previous state (`== 3`/`== 4`/`== 1`/`== 2`) to detect. Confirmed against
   the bot's C# reference (`PmzEntryStateMachine.IsFullReversal`, which reads the raw
   previous state) and against three unresolved Copilot PR review comments. Net effect: a
   full-reversal entry that should fire on the same 5-minute bar was silently delayed by one
   minute. Fixed by using the raw previous state throughout instead of the zeroed variable.

2. **"Secondary period cannot be less than primary" compile error.** `oneClose`/`fiveClose`/
   `fiveHigh`/`fiveLow` hard-coded `AggregationPeriod.MIN`/`FIVE_MIN` regardless of the chart's
   own aggregation setting — if the chart wasn't literally on the 1-minute timeframe, the study
   failed to even apply. Fixed by flooring to the chart's own primary period (the same pattern
   already used elsewhere in the file), which also means a misconfigured chart now shows the
   existing "use a 1-minute chart" warning banner instead of a raw compiler error.

3. **Every session boundary was platform-timezone-dependent.** `09:30`, `07:25`, `15:55`,
   `16:00`, and the `09:35` entry gate were all bare Eastern-Time literals compared against
   `SecondsFromTime`/`SecondsTillTime`, which read the *platform's displayed* clock, not the
   exchange's. On a non-Eastern platform (e.g. Central, the actual trader's setup) every
   boundary silently misaligned by the zone offset. Fixed by anchoring to
   `RegularTradingStart()`/`RegularTradingEnd()`/`GetTime()` (absolute epoch milliseconds,
   timezone-display-independent) instead — no operator configuration needed. This also
   upgraded the old exact-minute-equality resets (`SecondsTillTime(X) == 0`, fragile against
   any single missing bar at that exact minute) to crossed-the-boundary range checks.

4. **Root cause of "zero entries ever fire," even on days with clean price action:**
   `smState` is a recursive `CompoundValue(1, PmzEntryMachine(smState[1], ...).newCode, 0)`.
   `CompoundValue`'s seed only rescues bar 0 of the *entire* loaded chart history — if
   `smState` ever evaluated to `NaN` on any later bar (data gap, weekend/holiday transition),
   every subsequent bar inherited `NaN` via `smState[1]` forever, with no recovery. This
   silently disabled every entry-fire condition for the rest of the chart while all the
   per-bar diagnostics (gate, distance, above/below classification) stayed completely healthy
   — which is why the state machine looked fine in isolation but never fired. Found via
   hidden diagnostic plots (`DebugState` showing `N/A` while every other diagnostic on the
   same bar showed a real value). Fixed by guarding both `statePrev`/`armPrev`/`budgetPrev`
   extraction points to reset to a clean `0` on a `NaN` previous state instead of propagating
   it — confirmed live against a real Friday session in TOS.

Full commit-by-commit history is preserved in this repo (filtered from the original
`PeakBot-LLC/pmzbot` monorepo, path `Documentation/ASSETS/PMZBot_T1T3_EntryIndicator.ts.txt`).
