# Raptor Lake field notes

Field notes from tuning two i9-14900KS machines on the same small-form-factor Z790 board.
Written like a small encyclopedia: what we tried, what broke, what the numbers were, and what we learned.

The spirit is FAFO: fuck around and find out. Test your own system. Do not copy our numbers.
Every chip, board, kit and loop is different, and the only proof that counts is a test on your own hardware.

## The two machines

| | Projector | Printer |
|---|---|---|
| CPU | i9-14900KS, SP 101 (P-core 101, E-core 86) | i9-14900KS, SP 110 |
| Board | ASRock Z790I Lightning WiFi, BIOS 10.01, microcode 0x12F | same model, same microcode |
| Cooling | SuperCool direct-die hybrid frame, liquid metal, custom loop shared with the GPU | SuperCool direct-die frame, liquid metal, custom loop |
| RAM | Kingston FURY Renegade 2x24 GB DDR5-8400 CL40 (SK hynix 24 Gb M-die, CUDIMM) | 2x48 GB DDR5-6600 |
| GPU | RTX 5090 in the same loop | RTX 4070 SUPER |
| Role | everyday desktop, games, local AI | 24/7 home server |

Almost all measurements here are from Projector. Printer is the comparison point. See [hardware.md](hardware.md).

## How to read this

| File | What it covers |
|---|---|
| [hardware.md](hardware.md) | The platform, cooling, loop order, sensors, BIOS baseline, verified voltages |
| [cpu/method.md](cpu/method.md) | How CPU stability was tested and why |
| [cpu/results.md](cpu/results.md) | Every CPU run, per-core failure maps, voltage floors, E-core and ring tests |
| [cpu/thermals.md](cpu/thermals.md) | Core-to-water figures, GPU heat, loop plateau |
| [cpu/quirks.md](cpu/quirks.md) | Board and tooling surprises on the CPU side, load-line research |
| [ram/research.md](ram/research.md) | DDR5 tuning research, with sources |
| [ram/method.md](ram/method.md) | The unattended RAM sweep, plan v2, and what unattended operation cannot do |
| [ram/results.md](ram/results.md) | Every RAM step and its outcome, and the kept set |
| [ram/predictions.md](ram/predictions.md) | What the research predicted for each risky step, and what happened |
| [ram/quirks.md](ram/quirks.md) | DDR5 and BIOS surprises on the RAM side |
| [fafo-log.md](fafo-log.md) | Chronological log, including our own mistakes and corrections |
| [tools.md](tools.md) | The tools we built and how to reproduce the method |

Labels used throughout:

- **Measured**: read from a log on the machine.
- **Source**: a public source says so (link given).
- **Estimate**: our own arithmetic or reasoning.

## The FAFO principle, with safety rails

Fuck around: change one thing, test it hard, write down what happened.
Find out: believe the test, not the forum post, and not the first pass.

Rails we kept, and you should too:

1. **One change at a time.** Two changes and a pass tells you nothing about either.
2. **Know your way back before you start.** Save the working BIOS profile to a slot. Know where your Clear CMOS jumper is. Have a failsafe that restores the last good settings without you, and know what it cannot do: a no-POST or a hard freeze needs hands.
3. **Set hard stop limits before the run, not during it.** Ours: Vcore never above 1.360 V, coolant 45 C, and a stop on any error.
4. **A single pass is not stability.** Run it twice. Run it long. Run it hot. Run a control with nothing changed.
5. **Heat is part of the test.** A setting that passes on cold water can fail an hour later on warm water. A RAM setting that passes at 52 C can fail at 60 C. Test at the temperature real use reaches, and keep the CPU cool while you test the RAM.
6. **Check your files after crashes.** `chkdsk /scan` and `sfc /scannow` after any session with a crash. Unstable RAM can corrupt data silently.
7. **Distrust your own tooling.** Many of our "failures" were bugs in our scripts, and some of our "facts" were wrong. See [fafo-log.md](fafo-log.md).
8. **Voltage is a one-way street.** Raptor Lake degrades with voltage and heat (Intel's own statement). Lower clocks are free to undo. Lost silicon is not.
9. **A prediction is a guide, not a verdict.** Research told us five steps would fail. All five passed the short runs. Four still hold; the fifth (tREFI 65535) failed only in long runs with the hotter stick near 60 C. Nothing warned us about the one step that stopped the PC from showing a picture. See [ram/predictions.md](ram/predictions.md).
10. **Short runs to search, long runs at checkpoints.** We once stacked about 25 changes on 16-minute tests. The first long run then implicated all of them at once.

Nothing here is a recommendation to run any setting. Direct-die cooling, liquid metal, fixed voltages and memory overclocks all carry real risk of damage and data loss. You own the outcome.

## Status: 15 November 2026, morning

**The RAM tune is done for now.** The final set passed a three-hour memory validation on 15 November (04:06 to 06:55).
**The CPU re-tests after the liquid-metal repaste are in progress.** Nothing on the CPU side has changed yet.

### RAM: final set (validated)

Validation (Measured, DIMM fan on, no GPU heat): an at-rest test with 30 GiB held and checked every 2 minutes for 45 minutes (0 flips),
60 minutes of y-cruncher (30 sections, WHEA 0), and 60 minutes of TestMem5 0.13.1 with the anta777 config (81% of memory, hotter stick up to 62.0 C).

| Setting | XMP | Final | First value that failed |
|---|---|---|---|
| Speed | DDR5-8400, Gear 2 | same | 8600 not tried (no gain expected) |
| tCL | 40 | 38 | 36: corrupts memory at boot (waits for the DIMM water block) |
| tCWL | 40 | 34 | lowest tried |
| tRCD | 52 | 49 | 48: startup crash loop |
| tRCDW | 52 | 32 | lowest tried |
| tRP | 52 | 46 | lowest tried (the floor of the ladder) |
| tRAS | 132 | 64 | not binding below tRCD + tRTP |
| tRTP | 23 | 12 (entered 13) | 12 is the chip minimum. The BIOS shows the entered value minus 1 |
| tWR | 48 | 12 | lowest tried |
| tRRD_L | 12 | 11 | 10: startup crash loop |
| tRFC / tRFC2 | 922 | 660 | 650: test error when warm |
| tRFCpb | 796 | 420 | 400: blue screen at startup |
| tREFI | 32767 | 32767 | 49151 and 65535: TestMem5 errors at about 60 C on the hotter stick |
| tWRRD_dg / tWRRD_sg | 62 / 94 | 46 / 64 | 44: no train / 62: test error at once |
| tRDWR_dg / tRDWR_sg | 22 / 22 | 19 / 19 | 18: no train (each alone and as a pair) |
| tRDRD_sg | 16 (trained) | 15 | 14: no POST |
| tWRWR_sg | 16 (trained) | 15 | lowest tried |
| Command Rate | 2N | 2N | 1N: no train |
| TX (ASRock VDD_IMC) | 1.45 V | 1.35 V | |
| VDD2 (ASRock VDD_CPU) | 1.45 V | 1.45 V | 1.42 V: test error |
| VCCSA | 1.25 V | 1.25 V | not moved |
| DIMM VDD / VDDQ | 1.45 / 1.45 V | 1.50 / 1.48 V | needed for tCL 38 |

The biggest lesson of the RAM work: **the faults that were left were heat-bound.** tREFI 65535 passed every short run and both long y-cruncher runs,
then failed TestMem5 after 16 to 50 minutes with the hotter stick near 60 C. Looser refresh times (tRFC 688, tRFCpb 460) did not fix it. A shorter refresh interval did.
The two sticks run 6 to 9 C apart. See [ram/results.md](ram/results.md).

### What it bought (Measured)

| Metric | XMP | Final |
|---|---|---|
| Memory latency, our own pointer-chase meter | 75.2 ns | about 71 ns |
| Read bandwidth, same tool | about 127 GB/s | about 124 to 130 GB/s (run-to-run spread about 2 to 4 GB/s) |

The latency meter is ours. It is not AIDA64 and the numbers do not compare with AIDA64 numbers.
Bandwidth is "no worse", not a gain. The before-and-after benchmarks with every background load off are still owed.

### CPU

- Daily profile, unchanged: 57x all P-cores, E-cores 46x, ring 50x, fixed 1.340 V, LLC Level 1.
- E-cores 47x: hard freeze. Ring 51x: failed Cinebench with GPU heat at 1.340, 1.350 and 1.360 V (before the repaste).
- **Repaste, 14 November:** the liquid metal on the die, only 3 to 4 days old, had turned to a matt crust over the P-core area. A fresh syringe fixed it.
  Same 100-second all-core load: 430 W gave 95 C before, 440 to 513 W gives 81 to 83 C after. About 16 C per 100 W above water before, about 13 after. P-core spread 19 C before, 11 C after.
  Liquid metal does not dry out, it alloys; a crust days after the job points at the job. See [cpu/thermals.md](cpu/thermals.md).
- **Re-tests after the repaste, in progress:** 58x all-core at 1.340 V failed again on 15 November (Cinebench with GPU heat crashed at 2:11, one corrected error on P6, hottest core only 92 C, so not heat).
  1.350 V and 1.360 V are being tried next, then ring 51.

### Printer

- Memory, same tool: 103 to 106 ns idle, about 111 ns under light load, about 92 GB/s (DDR5-6600), against about 71 ns on Projector.
- Repaste on 14 to 15 November: hottest core from 14.0 to 12.7 C per 100 W above water, and a P7 hot spot gone.
- Printer tuning is next.

### Still to come

1. The CPU re-tests above (58x at 1.350 / 1.360 V, ring 51).
2. Before-and-after benchmarks against the original profile, with every background load off.
3. After the DIMM water block: tCL 36 and tREFI above 32767 get another try.
4. Printer: the same RAM and CPU method.

## Licence

The text in this repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE.md](LICENSE.md).
Linked sources belong to their authors.
