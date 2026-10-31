# SharedAFL contracts

Each `lib/AFL_Lib_*.afl` file documents its own `REQUIRES` / `PROVIDES` (or
signal contract) in the header. Read that header before including.

## Adapter pattern

Keep libs strategy-agnostic. Assign strategy variables to the names the lib
expects immediately before `#include`:

```afl
UniScore     = QualityScore;
UniLockBar   = RthOpen;
UniSnapValid = IsRTH;
UniCandidate = AvgDailyRthVol > 0;
#include "g:\OneDrive\01_TRADING\AFL_DEV\SharedAFL\lib\AFL_Lib_UniverseRank.afl"
```

## StrategyCode namespacing

Every static var used by the libs should be keyed by `StrategyCode` so two
strategies can run in one AmiBroker session without colliding.

## Absolute includes only

AmiBroker resolves relative `#include` paths against its Formulas directory,
not the including file. Always use absolute paths into SharedAFL.

## Trade context + custom backtest

After `DailyATR` is set and VwapBands + DailyContext have run:

```afl
#include "g:\OneDrive\01_TRADING\AFL_DEV\SharedAFL\lib\AFL_Lib_TradeContext.afl"
```

Snapshot those arrays at entry. On exit write a `|`-delimited payload keyed by
`StrategyCode + "Ex" + Name() + DateTimeToStr(exitDt)`. Set `CbtFieldNames` to
a comma list matching payload order (field 0 = Exit Reason, text; rest numeric).
Include the CBT lib at formula **top level** after signals are final:

```afl
CbtFieldNames = "Exit Reason,Entry Price,Avg Daily Vol,...";
#include "g:\OneDrive\01_TRADING\AFL_DEV\SharedAFL\lib\AFL_Lib_CustomBacktest.afl"
```

Leave `CbtFieldNames = ""` on Explore/score-only paths. Custom columns appear
only on **Portfolio** backtest. These metrics do not filter symbols — use them
to decide later hard filters.

## IbLogEvt — outermost include only

`IbLogEvt()`, `IbClampStopPx()`, and `IbStatusWorking()` live in
`AFL_Lib_IbLogEvt.afl`. `AFL_Lib_IBEngine.afl` calls all three. Every host
that includes IBEngine **must** `#include_once` `AFL_Lib_IbLogEvt.afl`
as the **first line of the outermost formula** (never only via a nested include,
and never inside `if`/`else` — AmiBroker rejects nested `function` defs and
ignores functions from nested `#include`s). Do **not** put `function` in
`AFL_Lib_IBEngine.afl`: SZDZ includes that file from inside `if (_isChart)`.

**Symptom if missing:** `Error 30. Syntax error, identifier 'iblogevt' is undefined`
on `IbLogEvt(...)` inside IBEngine. AmiBroker often reports this against a
stale `Formulas\Drag-drop\*.afl` chart copy, not the disk strategy file.

**Fix:** put `#include_once` of `AFL_Lib_IbLogEvt.afl` at line 1 of the file
actually applied to the chart (or Analysis formula). Then Formula Editor →
**Apply**, or close and reopen the pane. Editing the `.afl` on disk does not
update an already-open chart — AmiBroker keeps the cached host text.

`AFL_Lib_ChartOverlays.afl` reads host Chart Display Params for the GFX
level lines. `ShowStopLine` / `StopLineColor` / `StopLineWidth` are required
host Params (same rule as the VWAP band Params). Entry / target / last-price
(`ShowEntryLine`, `EntryLineColor`, `EntryLineWidth`, `ShowTargetLine`,
`TargetLineColor`, `TargetLineWidth`, `ShowCurrentPriceLine`,
`CurrentPriceLineColor`, `CurrentPriceLineWidth`) are optional: if the host
already `Param()`s them (GAG), the lib uses those values and does not
re-declare. If the host omits them (ORB), `typeof()` guards declare the
Params in the lib so Error 29 does not fire.

## IBEngine method 3 (pullback LMT) brackets

When `EntryMethodParam == 3` and stop distance is available
(`StopDistOverride` or ATR×`AtrStopMult`), `AFL_Lib_IBEngine.afl` places a
native IB bracket:

- parent LMT at `LmtPxLong` / `LmtPxShort` (`Transmit=False` while children attach)
- child STP at entry ± stop dist (`ParentID` + OCA)
- optional child LMT TP when `TargetDistOverride` > 0 and not VWAP-LMT exit mode
- last child uses `AutoTradeTransmit` so the whole bracket transmits together

Pending child OIDs are stored in `BrkStopOID{Long,Short}` /
`BrkTgtOID{Long,Short}` until fill promotes them to `StopOID` / `TargetOID`.
Fill protection therefore lives at the broker even if the chart is closed.

**Blackout:** cancel the unfilled parent LMT **and** its `Brk*` children.
Do not cancel `StopOID` / `TargetOID` for a position taken before the window.
A fill-race guard leaves parent+children intact when the parent status is
already `Filled` so fill detection can promote them.

Entry reject handling: `Error` / `Inactive` (e.g. IB **201** margin) clears
the OID after a status-guarded cancel, then **cooldowns** (or **halts** on
margin / 5-streak) so the chart does not re-spam brackets every refresh.
`RESET TRACKING` clears `EntryHalt`. Notional sizing caps use
`AvailableFunds` (not only `NetLiquidation`).
