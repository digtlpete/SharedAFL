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
