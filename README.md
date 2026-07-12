# PMZBot T1+T2+T3 Entry Indicator

A standalone ThinkOrSwim (ThinkScript) study for `/ES` continuous futures that visually
replicates the PMZBot trading bot's PMZ entry signal logic — without needing the bot running.
It plots the adjusted PMZ band, the ±distance admission gate, neutral markers, and CALL/PUT
entry triangles at the same bar the bot's live `PmzEntryStateMachine` would fire.

**File:** `PMZBot_T1T3_EntryIndicator.ts.txt`

## What it does

- Computes the premarket PMZ High/Low band from `/ES` continuous futures price action
  (07:25–09:30 ET), applying the same 5–8 point width clamp the bot uses
  (`PmzWidthAdjuster`-equivalent).
- Implements the same five-state pullback admission model as the bot's entry state machine:
  `InPMZ_NoPullback`, `InPMZ_CallPullback`, `InPMZ_PutPullback`, `OutsidePmz(Call)`,
  `OutsidePmz(Put)` — including the 09:35 ET entry gate, max-distance rejection, full-reversal
  arming, and pass-through reset.
- Plots a single CALL/PUT marker per admission (labeled to represent the bot's T1+T2+T3
  simultaneous tier fan-out — it does not simulate tier exits, slot occupancy, or which tiers
  are already in a trade).

## Explicitly out of scope

- Exit markers, consolidation pause, active-trade/cooldown/de-dupe awareness, tier-close
  simulation — entries only.
- SPX strike/premium selection, IV, DTE — this is a pure ES-price entry-timing visual, not a
  trade sizing tool.

## Setup in TOS

1. Chart: **TOS continuous `/ES`**, **1-minute**, with extended/pre-market hours enabled and
   at least a few days of lookback (the premarket accumulator needs a full prior session to
   seed correctly).
2. Studies → Edit Studies → Create → paste the full contents of
   `PMZBot_T1T3_EntryIndicator.ts.txt` → Apply.
3. Confirm the study is on the **price subgraph** (not a separate lower panel) — if PMZ/gate
   lines don't track the ES price axis, use the study's gear icon → Move to → Existing
   subgraph → Price.
4. Turn on `Banner` (input, off by default) for the full diagnostic label stack (`State`,
   `CleanArm`, `Budget`, `Gate`, `LIS`, `PMH/PML Time`, etc). Note: `AddLabel` only ever shows
   the **latest/current bar's** values, not whatever bar you've scrolled to — it does not
   follow the chart viewport or cursor.
5. To inspect a **historical** bar's internal state, hover the crosshair over that bar and
   check the Data Window/tooltip for the hidden `Debug*` plots (`DebugState`, `DebugCleanArm`,
   `DebugBudget`, `DebugGateOpen`, `DebugOneAbove`, `DebugOneBelow`, `DebugCallDist`,
   `DebugPutDist`) — unlike labels, plots are inspectable on any bar, not just the latest one.
   TOS may hide plot *names* next to their values by default; there's usually a per-study
   toggle (gear/eye icon near the study name) to show them.

## Known limitations / open items

- **Timezone:** session boundaries are anchored to `RegularTradingStart()`/
  `RegularTradingEnd()` (absolute epoch time), so they're correct regardless of the platform's
  displayed time zone. This does **not** yet extend to the non-default "ALL" premarket mode
  or the optional after-hours overlay (`futures` input, off by default) — those still compare
  raw ET clock literals against the platform's displayed clock and will misalign on a non-
  Eastern platform.
- **Validation:** parity against the bot's 23 `TestScenarios.cs` fixtures has only been
  checked analytically (code trace) for the full-reversal/cross-through scenarios (07, 08, 09,
  23) — a real TOS replay of the full scenario set, a side-by-side PNG comparison, and a live
  session spot-check against bot log timestamps are all still outstanding.
- The `tasks.md`-vs-script `PullbackConfirmation` input naming mismatch flagged in PR #217's
  review was never resolved — the script has no such input (it's always strict-pullback mode).

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
