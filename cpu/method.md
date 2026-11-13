# CPU method

## The question

What is the fastest P-core setting that is actually stable when the machine is hot, at or below 1.360 V?

The owner's success test: one Cinebench pass, then a 10-minute Cinebench run, both heat-soaked.
We later added 30-minute runs, because 10 minutes turned out not to be enough (see below).

Pass means all of:

- the load finishes with no crash, freeze or error dialog,
- zero corrected CPU errors (WHEA) in the run,
- no thermal throttling, and the measured clock equals the set clock on every core.

## The symptom we started from

A review of 120 days of the Windows event log before tuning (measured):

- 116 corrected CPU errors, all in MCA bank 0 (the core's instruction-fetch front end).
- Two types: instruction TLB (MCA code 0x0010) x78, internal parity (0x0005) x38.
- By core: P2 47, P4 37, P3 15, P6 12, P5 4, P7 1. P0 and P1: zero.
- They came in bursts lasting minutes, which matched benchmark runs, not idle use.

These are per-core margin faults. The public fixes are more voltage or less clock on that core
(for example the [overclock.net 14900KS thread](https://www.overclock.net/threads/14900ks-oc-help.1811739/) and
[this r/overclocking thread](https://www.reddit.com/r/overclocking/comments/1cqdu9z/14900k_whea_parity_error_on_power_limited_and/)).
Voltage was capped at 1.360 V by the owner, so clock had to give.

## Tools

- **Shakedown**: our own Windows tool. It runs the load, watches the WHEA log and maps every error to a core,
  logs per-core clocks and temperatures, closes the load when it crashes, and writes a verdict to a ledger.
  It also has a **GPU heat** switch: a compute shader that holds the RTX 5090 at about 520 to 535 W.
  See [tools.md](../tools.md).
- **Cinebench 2026.1.3**, the standalone build, multi-core, driven from the command line.
- **ASRock A-Tuning**: changes the P-core ratio and the fixed Vcore live from Windows on this board.
  We drove it with a small automation helper (see [quirks.md](quirks.md)).

## Why Cinebench 2026 with GPU heat

- Cinebench is the owner's reference load. It runs all cores at the full ratio and draws about 400 to 420 W here.
- On this machine it reproduced the logged fault on demand: the same cores (P2 first, then P4) failed within seconds to minutes.
- The CPU and GPU share one loop. In real use (games, AI work) the GPU dumps 300 to 600 W into the water.
  A CPU test with an idle GPU tests a colder machine than the one you actually use.
- GPU heat at about 530 W drives the water from about 19 C to 30 C in under 5 minutes (about 2.3 C per minute) and then holds a plateau.

Cinebench is not a heavy AVX2 load and it is not a light single-thread load.
It does not prove stability in Prime95 small FFT, OCCT AVX2 or shader compilation. Run those as well if they matter to you.

## Heat soak

"Heat-soaked" here means the loop has reached its plateau under CPU plus GPU load.
On Projector that is about 36.5 to 39.5 C at the pump with about 950 W in the loop and fans at 60 to 65%.

Water temperature changes **how soon** a marginal setting fails, not **whether** it fails:

- At 59x and 1.360 V, a single 2.5-minute pass on 18 C water passed.
- The same setting from 18.6 C water failed at 3:34 (P2, core at 72 C, water 21.8 C).
- At 58x, +20 mV bought about 5 C of core temperature on P2 before the first error.

So a fair test starts warm, or runs long enough to get warm.

## Why 10-minute runs overstate the voltage floor

The 10-minute sweep at 57x said the floor was 1.310 V (1.300 V failed).
The 30-minute runs said otherwise:

| Voltage | 10 min | 30 min |
|---|---|---|
| 1.320 V | PASS | FAIL (1 P2 error at 26:13) |
| 1.330 V | PASS | PASS, then a crash at 15:24 with no WHEA record |
| 1.340 V | PASS | PASS x3 |

The real floor at this heat was about 30 mV above the 10-minute floor. Reasons, as far as we can tell:

1. **Error rate, not a cliff.** Near the floor, errors are rare events. At 1.320 V we saw about one error per hour of load. A 10-minute run catches that only about one time in six.
2. **The loop keeps warming.** Over the night the plateau crept from 36.5 to 39.5 C and the hottest core rose from about 92 to 100+ C. Later runs are harsher than earlier ones.
3. **A crash with no WHEA record is still a fail.** At 1.330 V the load died silently. Count those.

Our rule now: find the floor with short runs, then validate with 3 x 30 minutes back to back, hot, and add margin above the voltage that passed.

## Control runs and order

- One change per run. Ratios and voltage are never changed together.
- When a run fails, repeat the last passing setting before blaming the new one. If the control fails too, the base was marginal, not the step.
- Keep back-to-back runs back to back. A cooldown between runs makes later steps look better than they are.
- Write the line "testing X" to the journal before each step. If the PC locks up, the last line says what was running.

## Sequence we used

1. Baseline at 59x all-core, 1.360 V, to see which cores fail.
2. Per-core ranking: hold the failing cores out of the load (job affinity limit) and run again, until it passes.
3. Per-core ratios in the BIOS (Specific Per Core).
4. All-core 58x, then 57x, live through A-Tuning.
5. Voltage sweep at 57x, 10 mV steps down, 10 minutes each with GPU heat, no cooldown.
6. Overnight validation: 3 x 30 minutes per voltage from 1.320 V up, stop at the first voltage with 90 clean minutes.
7. Hybrid per-core profiles at the validated voltage (P2 at 57x, the rest 58x) as a last check.
8. Final profile written to the BIOS and confirmed by a SCEWIN export.
