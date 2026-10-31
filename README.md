# SharedAFL — reusable AmiBroker AFL plumbing

Canonical shared libraries extracted from the ORB strategy. New strategies
should `#include` these with **absolute paths** (AmiBroker resolves relative
includes against its Formulas folder, not the including file).

## Layout

```
SharedAFL/
  lib/                  AFL_Lib_*.afl plumbing modules
  Strategy_Template.afl thin harness — copy, rename StrategyCode, replace SIGNAL
  docs/                 contracts and usage notes
  README.md
```

## Libraries

| File | Role |
|------|------|
| `AFL_Lib_SessionTime.afl` | RTH session math, day boundaries, time filters |
| `AFL_Lib_VwapBands.afl` | RTH VWAP + SD bands, prior-day references |
| `AFL_Lib_DailyContext.afl` | daily EMAs, Average Daily RTH Volume |
| `AFL_Lib_TradeContext.afl` | gap %, PD Open In1SD, EMA/ATR distances (needs DailyATR) |
| `AFL_Lib_CustomBacktest.afl` | portfolio trade-list custom columns (pipe payload) |
| `AFL_Lib_UniverseRank.afl` | daily Top-N cross-sectional gate (adapter inputs) |
| `AFL_Lib_PositionSizing.afl` | `SizingRiskQty()` / `SizingFinalQty()` (scalar) |
| `AFL_Lib_RealizedPL.afl` | static-var realized P/L tracking |
| `AFL_Lib_IbLogEvt.afl` | `IbLogEvt()` helper (include at formula top level) |
| `AFL_Lib_ChartOverlays.afl` | band plots, markers, GFX info box (needs IbLogEvt) |
| `AFL_Lib_IBEngine.afl` | live IB execution behind a documented signal contract (method 3 = native IB brackets: parent LMT + child STP/TP) |

**IbLogEvt:** `#include_once` `AFL_Lib_IbLogEvt.afl` as the first line of the
outermost formula (the file applied to the chart), then re-Apply. Nested
includes do not register the function. See
[`docs/contracts.md`](docs/contracts.md#iblogevt--outermost-include-only).

## Include path

```afl
#include "g:\OneDrive\01_TRADING\AFL_DEV\SharedAFL\lib\AFL_Lib_SessionTime.afl"
```

Optional: junction `AmiBroker\Formulas\Include\SharedAFL` → this `lib\` folder
so formulas appear in AmiBroker's browser while Git still owns the source.

## New strategy workflow

1. Copy `..\AFL_DEV\_StrategyScaffold` to a new folder under `AFL_DEV` and
   rename it (e.g. `MyIdea`). Run `git init` in the copy.
2. Set a unique `StrategyCode` in `Strategy.afl` (namespaces static vars).
3. Satisfy each lib's `REQUIRES` / signal contract before the `#include`.
4. Replace only the SIGNAL section.
5. Generate and run backtests via the scaffold's `tools/` harness (see
   `_StrategyScaffold\README.md`).
6. Before accepting any SharedAFL change, re-run the ORB regression gate
   (`RESULT: IDENTICAL`) from the ORB repo.

## Compat shims

`ORB\lib\ORB_Lib_*.afl` are thin redirects into this tree so old absolute
paths keep working. Prefer `AFL_Lib_*` in new code.

## Merge bar for lib changes

Treat SharedAFL like a product: small changes, header contracts intact, and
the ORB golden trade-list regression still green before merging.
