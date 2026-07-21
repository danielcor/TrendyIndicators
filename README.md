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
| `PMZBot_T1T3_EntryIndicator_Auto.ts.txt` | **Day *and* night** auto WIP (allowlist DayOnlyMode/Force, DistTpl). Prefer `_Auto_RC_DistTpl` when overnight must stay solid. |
| `PMZBot_T1T3_EntryIndicator_Auto_RC_DistTpl.ts.txt` | **Preferred Auto RC** — known-good night math (`c905043` + alerts) + `ShowLiveEntryLabel` + `UseInstrumentDistanceDefaults` (normal PMZ, family maxD ES/MES=10 RTY=5 NQ=25 YM=25 CL/MCL=20) + continuous family night (ES/MES NQ/MNQ RTY/M2K/TF YM/MYM CL/MCL) + optional `AggressiveEntries` (each 5m outside in range fires — no neutral arming, no same-side took block; default no). |
| `PMZBot_T5_ContinuationIndicator.ts.txt` | **Tier5 only** — standalone Trendy Cloud + 1m velocity continuation markers (not PMZ SM). Stack on the same price subgraph as a T1–T3 study. |

## Entry markers & profit targets

All 4 study files share the same entry-marker inputs:

- `ShowEntryMarkers` (default yes): green/red triangle at the 1-minute close when a CALL/PUT
  entry fires.
- `ShowProfitTargets` (default yes): 3 small circles at the entry bar only, at
  `entry close ± ProfitTarget1/2/3Points` — green for CALL, red for PUT. Static reference
  levels, not live "target hit" tracking or exits. Implemented as separate
  `ProfitTarget1/2/3Call` and `ProfitTarget1/2/3Put` plots (6 per study) with fixed colors —
  a single shared plot colored via `AssignValueColor` was unreliable for sparse POINTS plots.
- `ProfitTarget1Points` / `ProfitTarget2Points` / `ProfitTarget3Points` (defaults 5 / 10 / 15):
  configurable point deltas from the entry close for each target circle.

The Auto study also has **optional alerts** (all default **off**):

- `AlertOnEntry`: Message Center text (`PMZ BUY/SELL entry @ level`) when an entry fires.
- `AlertEntrySound`: optional `Sound.Ring` on entry alerts. Off = silent text only (still needs `AlertOnEntry`).
- `AlertOnProfitTarget`: first touch of each TP level after an entry — one Message Center alert per target per entry (latches reset on each new entry; touches count from the bar after the entry bar). BUY touch = bar high ≥ target; SELL touch = bar low ≤ target.
- `AlertProfitTargetSound`: optional sounds for TP alerts (`TP1` Ding, `TP2` Chimes, `TP3` Bell). Off = silent text only (still needs `AlertOnProfitTarget`).
- Platform: alerts fire on live bars only (never historical backfill) and land at 1-minute bar completion (`once_per_bar`). To hide Message Center and keep sound only: study gear → Alerts → uncheck "Show in Message center".

The Auto study additionally shows a latched **live-entry label** (top-left, under the compact
PMZ High/Low labels): exactly one at a time, always the most recent entry — `BUY/SELL HH:MM TZ
@ level | TP t1 / t2 / t3`, green for BUY, red for SELL. Gated by `ShowStatusStrip`.

Note (Auto study): the combined `callFire`/`putFire` gates NaN-guard each session stream before
`or`-combining. Outside its own session a stream's fire signal is NaN, and thinkScript `or`
does not short-circuit — unguarded, the combined gate is NaN on every bar and anything gated
on it silently never renders.

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

## Auto (day and night)

`PMZBot_T1T3_EntryIndicator_Auto.ts.txt` runs **two** entry state machines:

- **Day** in cash RTH on **every** symbol (equity and futures).
- **Night** only when all of the following hold:
  1. Chart symbol is in the **CME US equity-index schedule family** (same cash RTH + Globex calendar as `/ES`), and
  2. `DayOnlyMode = no` (default).

### Inputs unique to Auto

| Input | Default | Role |
|-------|---------|------|
| `DayOnlyMode` | `no` | When **yes**, suppress night PMZ band + night SM/entries on **all** symbols (day cash path only). Use this on `/ES` when you want the Auto study without overnight markers. On equity/unsupported futures, set **yes** to silence the allowlist warning. |
| `ForceAutoNightFutures` | `no` | Force Auto night when the chart is a **dated** front month (e.g. `/CLU25:XNYM`) or another symbol not on the continuous equality list. |
| `ShowAutoAllowlistWarning` | `yes` | When `DayOnlyMode=no` and the chart symbol is **not** on the Auto day+night futures allowlist (equity or e.g. `/GC`), show high-visibility TOP_RIGHT warnings listing supported roots. |
| `AlertOnEntry` / `AlertEntrySound` | `no` | Optional entry Message Center (+ optional Ring). |
| `AlertOnProfitTarget` / `AlertProfitTargetSound` | `no` | Optional first-touch TP alerts (+ optional sounds). |

No `NightMode`, `Premarket`, or `futures` inputs. Still has clamp min/max, distance, budget, banners.

### Symbol support

| Chart | Day band + day entries | Night band + night entries |
|-------|------------------------|----------------------------|
| **Equity** (SPY, QQQ, IWM, DIA, AAPL, …) | yes | no |
| **Allowlisted futures** (below) with `DayOnlyMode=no` | yes | yes |
| Same allowlist with `DayOnlyMode=yes` | yes | no |
| Other futures (GC, 6E, ZB, BTC, NKD, EMD, …) | day path may still paint if RTH epochs exist | **no** |

**Auto day+night futures allowlist** (continuous, dated front-month, micros; any exchange suffix on `GetSymbol()`):

| Root | Micros / aliases matched | Notes |
|------|--------------------------|--------|
| ES | MES | Equity-index canonical |
| NQ | MNQ | Equity-index |
| RTY | M2K, legacy TF | Equity-index |
| YM | MYM | Equity-index |
| CL | MCL | WTI crude — **included because upstream Tr3ndy/customer PMZ is used on /CL**; same ES-style session offsets, retune clamp/distance/TPs for $/bbl |

Examples that enable night when `DayOnlyMode=no`: continuous `/ES:XCME`, `/MES:XCME`, `/NQ:XCME`, `/MNQ:XCME`, `/RTY:XCME`, `/M2K:XCME`, `/YM:XCBT`, `/MYM:XCBT`, `/CL:XNYM`, `/MCL:XNYM`. Dated e.g. `/ESU25:XCME`, `/CLU25:XNYM` need `ForceAutoNightFutures=yes`.

Detection uses **exact** `GetSymbol()` equality on continuous roots (with and without exchange namespace). ThinkScript on TOS rejected regex `matches` here (`Invalid statement: def`). Dated front months are **not** auto-detected — set `ForceAutoNightFutures=yes`.

**Warning behavior:** with `DayOnlyMode=no` (default Auto), equity and non-allowlisted futures still run the **day** path, but orange/yellow chart labels warn that night is disabled and print the supported list. Silence with `DayOnlyMode=yes` or `ShowAutoAllowlistWarning=no`. For dated `/CLU25` etc., use `ForceAutoNightFutures=yes`.

### Known issues / limits of “all same-schedule futures”

1. **Allowlist only — not every futures product.** Night math uses the **US cash-style close tail → Globex ~18:00 reopen → next cash-style open** pattern (same code path as `/ES`). `/CL`/`/MCL` are allowlisted for customer/Tr3ndy PMZ use on oil; metals (`/GC`), FX (`/6E`), rates (`/ZB`), crypto, and Nikkei (`/NKD`) stay **day-only**.
2. **Point-scale clamp defaults are ES-centric.** `MinPmzWidthPoints`/`MaxPmzWidthPoints` default **5–8** (bot ES parity). NQ/MNQ, RTY/M2K, YM/MYM, and especially **CL/MCL ($/bbl)** need **different** clamp widths; leave `AdjustPMZ` on and tune per symbol or turn clamp off. Distance gate `MaxDistanceOutsidePmzPoints` (default 10) is likewise ES-scaled.
3. **Profit-target point defaults (5/10/15)** are also ES-scaled; retune on NQ/YM/RTY/CL/micros.
4. **`RegularTradingStart`/`End` on equities vs futures.** Day anchors come from TOS session epochs for the chart symbol. On `/CL`, those epochs may not match equity cash 09:30–16:00 exactly — verify with Banner / `DebugDayWindow` on live oil charts.
5. **Extended-hours chart required** for day premarket PMZ seed and for night form/display. Without extended hours, equity premarket and Globex overnight bars are missing.
6. **Dated roll / continuous vs front month.** Continuous (`/ES`, `/CL`) and dated (`/ESU25`, `/CLU25`) both match, but **PMZ levels are local to the chart series** — continuous vs front-month can differ around rolls.
7. **Composite / multi-leg symbols** (`/ES+/YM`, spreads): `GetSymbol()` / `GetSymbolPart` shape is not specially handled; treat as unsupported for night.
8. **Mid-cap / other equity index** (e.g. EMD) is **not** in the allowlist — add only after a product decision + live TOS check.
9. **Holiday Globex afternoon breaks** are still not calendared (same as before).
10. **No regex symbol detect:** TOS rejected `matches` in this study (`Invalid statement: def`). Continuous roots use exact equality; dated months need `ForceAutoNightFutures`.

Equity charts never arm night entries (no Globex session on the equity tape in this study). Allowlisted futures get day markers in the day window and night markers overnight unless `DayOnlyMode=yes`.

## Tier5 / MES momentum setups (standalone)

`PMZBot_T5_ContinuationIndicator.ts.txt` — **second study**, /MES (or /ES) **trade markers**.

- **Goal:** Clear one-lot futures P&amp;L path (not SPX option T5 parity).
- **Defaults (v2.1 MES):** `Profile=MES_BALANCED` (RTH, vr 0.60, cd 30, hold 5, hard 8, cloud 3, trail 5).
  Other presets: **MES_STRICT**, **MES_ACTIVE**, **MES_GLOBEX**, **SCOUT_DEBUG**, **CUSTOM**.
- **Markers:** Green **MES BUY @ price** / red **MES SELL @ price**; exit bubbles
  **X LONG/SHORT ±pts ±$** with reason (HARD/CLOUD/TRAIL/FLIP/FLAT).
- **Lines:** White entry, red stop while open; optional bar tint while in trade.
- **Stamp:** `MES T5 v2.1 | BALANCED|… | RTH|GLOBEX | SETUPS|SCOUT | vr>=… | stop …pt`
- **Stack** on `/ES` or `/MES` 1m price subgraph. `MesDollarsPerPoint=5` (MES) or `50` (ES).

## Explicitly out of scope

- PMZ T1–T3 exit/consolidation/slot simulation on the PMZ studies — entries only there.
- SPX strike/premium / full TrendyPlus option P0–P2.
- Holiday Globex afternoon-break calendar.

## Setup in TOS

1. Chart: **1-minute** with extended/pre-market hours, multi-day lookback.
   - Auto night: allowlisted futures (`/ES`, `/MES`, `/NQ`, `/MNQ`, `/RTY`, `/M2K`, `/YM`, `/MYM`, `/CL`, `/MCL`, continuous or dated).
   - Auto day-only / equities: SPY, QQQ, etc., or futures with `DayOnlyMode=yes`.
   - Day-only studies: still typically **`/ES`** continuous for bot parity checks.
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
