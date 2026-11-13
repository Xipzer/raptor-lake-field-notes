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
| RAM | Kingston FURY Renegade 2x24 GB DDR5-8400 CL40 (SK hynix 24 Gb M-die, CUDIMM) | DDR5-6600 |
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
5. **Heat is part of the test.** A setting that passes on cold water can fail an hour later on warm water.
6. **Check your files after crashes.** `chkdsk /scan` and `sfc /scannow` after any session with a crash. Unstable RAM can corrupt data silently.
7. **Distrust your own tooling.** Many of our "failures" were bugs in our scripts, and some of our "facts" were wrong. See [fafo-log.md](fafo-log.md).
8. **Voltage is a one-way street.** Raptor Lake degrades with voltage and heat (Intel's own statement). Lower clocks are free to undo. Lost silicon is not.
9. **A prediction is a guide, not a verdict.** Research told us five steps would fail. All five passed. Nothing warned us about the one step that stopped the PC from showing a picture. See [ram/predictions.md](ram/predictions.md).

Nothing here is a recommendation to run any setting. Direct-die cooling, liquid metal, fixed voltages and memory overclocks all carry real risk of damage and data loss. You own the outcome.

## Status: 13 November 2026, late afternoon

**The RAM sweep is still running.** The figures below are from the journal at 17:00. They will change.
Nothing in the kept set has had its long validation yet. Each kept value has two passing 7-minute stress runs, no more.

### CPU

- Daily profile: 57x all P-cores, ring 50x, fixed 1.340 V, LLC Level 1. Validated on 12 November with 3 x 30 minutes of Cinebench 2026 plus GPU heat, zero errors.
- **E-cores 46x kept** (was 45x). It passed two memory stress runs and 10 minutes of Cinebench with GPU heat.
- **E-cores 47x: hard freeze** about 3 minutes into the memory stress test. It needed the power button.
- **Ring 51x: fails** Cinebench with GPU heat at 1.340 V (2:24) and at 1.350 V (4:13). The ring stays at 50x.

### RAM, kept set so far

| Setting | XMP | Kept so far |
|---|---|---|
| Speed | DDR5-8400, Gear 2 | same |
| tCL / tCWL | 40 / 40 | 38 / 36 |
| tRCD / tRP | 52 | 49 |
| tRAS | 132 | 76 |
| tRFC / tRFC2 | 922 | 660 |
| tRFCpb | 796 | 420 |
| tREFI | 32767 | 65535 |
| tWR | 48 | 20 |
| tRRD_L | 12 | 11 |
| tWRRD_dg / tWRRD_sg | 62 / 94 | 48 / 70 |
| tRDWR_dg / tRDWR_sg | 22 / 22 | 19 / 19 |
| tRDRD_sg | 16 (trained) | 16 (trained, left alone) |
| TX (ASRock VDD_IMC) | 1.45 V | 1.35 V |
| VDD2 (ASRock VDD_CPU) | 1.45 V | 1.45 V |
| VCCSA | 1.25 V | 1.25 V |
| DIMM VDD / VDDQ | 1.45 / 1.45 V | 1.50 / 1.48 V |

### What it bought (measured, all provisional)

| Metric | Before | Now |
|---|---|---|
| Memory latency, our own pointer-chase meter | 75.2 ns (XMP) | about 70.5 to 71 ns |
| Read bandwidth, same tool | about 127 GB/s | about 128 to 130 GB/s in the latest runs |
| Cinebench 2026 multi, 10 minutes with GPU heat | about 9,875 to 9,885 (E 45x) | 10,059.55 (E 46x, tuned RAM) |

The latency meter is ours. It is not AIDA64 and the numbers do not compare with AIDA64 numbers.
Bandwidth moves by about 2 GB/s from run to run, so treat that row as "no worse".

### Still to come

1. Discovery and the extras pass are nearly done.
2. tRTP. It is still at its XMP value of 23. The first attempt (19) read back as 18 and was not tested.
3. A tRAS re-check. 72 is under test now (run 1 of 2 passed). The floor is tRCD + tRTP.
4. Retries of the two old crashes, tRCD/tRP 48 and tRRD_L 10, now that 49 and 11 pass.
5. Final validation: 60 minutes of y-cruncher, 30 minutes of Cinebench with GPU heat, three retrain boots.
6. Before and after benchmarks.
7. A short re-run of the heat-limited items after the DIMM water block is fitted.

## Licence

The text in this repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE.md](LICENSE.md).
Linked sources belong to their authors.
