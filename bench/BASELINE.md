# Wall-Time Performance Baseline — as of `bb3823a`

Recorded 2026-06-10 against the `--release --features jit` binary
(`target/release/Cx_0V.exe`), per the perf-rider dispatch. The matrix measures
correctness; this file is what "fast" means at the start of the D1 arc, so
every later arc answers "did we get slower?" with a number.

**Machine:** AMD Ryzen 7 5825U (laptop — thermally sensitive; expect the noise
band below, not single-digit-ms stability).
**Methodology:** `bench/run_bench.ps1` — PowerShell `Measure-Command`, full
process wall time, 1 discarded warmup + 7 timed runs per program per backend,
median reported, min..max spread alongside. The warmup run doubles as a
correctness screen (cross-backend stdout equality; any exit other than 0/127
is a crash-stop). Two full script runs are recorded raw — reproducibility is
part of the baseline.

## Recorded runs

### Run A

| program | interp median (ms) | interp spread | jit median (ms) | jit spread |
|---|---|---|---|---|
| bench_arith_loop | 1350 | 1,333..1,519 | 111 | 94..124 |
| bench_array_loop | 1427 | 1,309..1,760 | 112 | 103..118 |
| bench_fib_iter | 1745 | 1,711..1,905 | 126 | 108..149 |
| bench_fib_rec | 1372 | 1,296..1,447 | 112 | 102..122 |
| bench_if_expr | 1709 | 1,625..1,795 | SKIP | - |
| bench_str_concat | 1313 | 1,250..1,425 | SKIP | - |
| bench_struct_loop | 1637 | 1,481..1,741 | 118 | 98..121 |
| bench_when_loop | 1423 | 1,304..1,516 | 116 | 61..163 |

### Run B

| program | interp median (ms) | interp spread | jit median (ms) | jit spread |
|---|---|---|---|---|
| bench_arith_loop | 1412 | 1,337..1,521 | 120 | 111..135 |
| bench_array_loop | 1397 | 1,344..1,453 | 115 | 110..155 |
| bench_fib_iter | 1744 | 1,641..2,035 | 113 | 104..126 |
| bench_fib_rec | 1484 | 1,453..1,592 | 109 | 101..120 |
| bench_if_expr | 1791 | 1,715..1,837 | SKIP | - |
| bench_str_concat | 1344 | 1,265..1,387 | SKIP | - |
| bench_struct_loop | 1655 | 1,554..1,768 | 124 | 115..171 |
| bench_when_loop | 1376 | 1,281..1,491 | 107 | 102..117 |

## Noise estimate (observed, not invented)

- **Interpreter:** cross-run median drift observed up to **8.2%**
  (`bench_fib_rec` 1372 → 1484 — the one program whose two medians each fell
  just outside the other run's min..max band; all seven others reproduced
  inside the bands). Within-run spread up to ~±13% (`bench_if_expr`, run A).
- **JIT:** medians 107–126 ms, dominated by process startup + Cranelift
  compile, not loop work. Cross-run drift ≤ ~13 ms.

**Regression rule derived from the above:** a median drift beyond **±10 %**
(interpreter) or beyond **±20 ms** (JIT) that *reproduces across two
consecutive script runs* is a finding — reported, diagnosed, ruled on; never
silently absorbed. One run outside the band is weather; two is a regression
signal.

## Standing rule

Re-run `bench/run_bench.ps1` at each arc boundary (post-D1.1, post-D1.2,
post-gate, post-each-D2.x) and compare against this table. When a D-step
lands new lowering, SKIP rows flip to two-backend automatically (the runner
detects JIT support per program) — record the first JIT median for a newly
lowered program as that program's new JIT baseline.

## Surface notes

- Two programs are interpreter-only by design (`bench_str_concat` — Str has
  no IR; `bench_if_expr` — the #046 SKIP class). Still regression-tracked.
- `bench_array_loop` carries a shape note: array index-reads inside larger
  expressions (`acc + a:[k]`) do not lower today — the Index node's semantic
  type poisons the enclosing Binary as Unknown and the JIT SKIPs (perf-rider
  finding, filed for the D1 lowering queue). The benchmark routes element
  reads through typed locals; loads/stores still execute every iteration.
- Debug-vs-release output screen: all 8 programs produce identical stdout and
  exit codes on debug and release builds, both backends (screened at
  recording time; first time this had ever been checked).
