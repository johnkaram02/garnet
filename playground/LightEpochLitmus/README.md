# LightEpoch store-buffer litmus harness

Runs the real `LightEpoch` use-after-free race on hardware: a reader announces its
epoch and dereferences a page while a reclaimer retires that same page. Standalone
executable, `LightEpochLitmus`.

## Workload

A buffer pool is allocated in userspace up front and pages are handed out from it. A
page is never returned to the OS; "freeing" one stamps a poison sentinel over it. A
reader that sees poison in a page it was protecting is a use-after-free.

## Running it

```
dotnet run -f net10.0 --project playground/LightEpochLitmus -- --seconds 600 --json result.json
```

The control runs first and the harness refuses to continue unless the detector
reports it. `--help` lists every option. Exit codes: `0` pass, `1` violation, `2`
inconclusive, `3` unsupported host, `64` bad arguments. `0` versus `2` is the point —
an inconclusive run is not a pass.

In Docker, with the repository root as the build context:

```
docker build -f playground/LightEpochLitmus/Dockerfile -t garnet-lightepoch-litmus .
docker run --rm garnet-lightepoch-litmus --seconds 3600 --iterations 8 --json -
```

## Comparing against the unfixed algorithm

`helpers/BuggyLightEpoch.cs` is a frozen copy of `LightEpoch`.
`--buggy` runs against it, so both algorithms can be compared on the same machine in
the same session:

```
dotnet run -f net10.0 -c Release --project playground/LightEpochLitmus -- --buggy --seconds 30
dotnet run -f net10.0 -c Release --project playground/LightEpochLitmus -- --seconds 30
```

The first is expected to exit `1` with violations, the second `0`.

## Drain litmus: stranded actions on epoch release

`--drain` tests the **checked-out production `Tsavorite.core.LightEpoch`**. The client
compiles that same source under its own namespace. There is no substituted volatile
implementation in the stress run: checking out the pre-fix or post-fix commit changes
the behavior exercised by the identical command:

```powershell
dotnet run -f net10.0 -c Release --project playground\LightEpochLitmus -- --drain --seconds 180
```

Rebuild on each checkout; do not use `--no-build`.

Two pinned threads enter the epoch. One suspends after a swept jitter delay. The other
calls `BumpCurrentEpoch(action)` and suspends. Once both have returned, the action must
have run. A third rendezvous prevents the next round's `Resume()` from rescuing a
pending action before it is counted.

With `Volatile.Write` in `Release()`, the slot clear can remain buffered while the
leaving thread reads `drainCount == 0`. The bumper can increment `drainCount`, still
see that slot occupied, and also return without running the action. With no further
epoch activity, the action remains stranded indefinitely. A flush callback can
therefore leave socket requests unsent; this harness tests the underlying epoch
failure without requiring networking.

The next round deliberately rescues stranded actions so the harness can count
multiple occurrences instead of hanging forever. `stranded > 0` exits `1`.
`Interlocked.Exchange` on the slot clear closes this store-to-load window; a clean,
sampled run exits `0`. A finite clean run is evidence, not proof: on native x64,
the checked-out volatile-only release stranded 4 actions in 37.7M rounds over 180s,
after a separate 60s run observed none.
Use longer durations or `--iterations` if necessary.

The control runs first. It generates an epoch copy whose `Suspend()` reads
`drainCount` before clearing the slot, with a bounded 64-iteration `Thread.SpinWait`
to widen that calibrated race window without scheduler yielding. The build fails
if the edit no longer matches. The control must strand an action or the run exits
`2` (inconclusive). `--self-test` runs only the control, and `--no-control` skips it.
A clean stress result is also inconclusive unless both `drainedBySuspender` and
`drainedByBumper` are nonzero. These counters establish that both execution orders
were exercised; they cannot prove that every possible race was sampled.

## Do not run this under emulation

QEMU — `--platform linux/arm64` on an x86 host, say — does not reproduce the emulated
architecture's memory ordering. The harness checks for emulation at startup and refuses
to run.
