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
| [hardware.md](hardware.md) | The platform, cooling, loop order, sensors, BIOS baseline |
| [cpu/method.md](cpu/method.md) | How CPU stability was tested and why |
| [cpu/results.md](cpu/results.md) | Every CPU run, per-core failure maps, voltage floors |
| [cpu/thermals.md](cpu/thermals.md) | Core-to-water figures, GPU heat, loop plateau |
| [cpu/quirks.md](cpu/quirks.md) | Board and tooling surprises on the CPU side |
| [ram/research.md](ram/research.md) | DDR5 tuning research, with sources |
| [ram/method.md](ram/method.md) | The unattended RAM sweep and its safety nets |
| [ram/results.md](ram/results.md) | Every RAM step and its outcome |
| [ram/quirks.md](ram/quirks.md) | DDR5 and BIOS surprises on the RAM side |
| [fafo-log.md](fafo-log.md) | Chronological log, including our own mistakes |
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
2. **Know your way back before you start.** Save the working BIOS profile to a slot. Know where your Clear CMOS jumper is. Have a failsafe that restores the last good settings without you.
3. **Set hard stop limits before the run, not during it.** Ours: Vcore never above 1.360 V, coolant 45 C, and a stop on any error.
4. **A single pass is not stability.** Run it twice. Run it long. Run it hot. Run a control with nothing changed.
5. **Heat is part of the test.** A setting that passes on cold water can fail an hour later on warm water.
6. **Check your files after crashes.** `chkdsk /scan` and `sfc /scannow` after any session with a crash. Unstable RAM can corrupt data silently.
7. **Distrust your own tooling.** Several of our "failures" were bugs in our scripts. See [fafo-log.md](fafo-log.md).
8. **Voltage is a one-way street.** Raptor Lake degrades with voltage and heat (Intel's own statement). Lower clocks are free to undo. Lost silicon is not.

Nothing here is a recommendation to run any setting. Direct-die cooling, liquid metal, fixed voltages and memory overclocks all carry real risk of damage and data loss. You own the outcome.

## Status

- **CPU: done.** Daily profile on Projector: 57x all P-cores, E-cores 45x, ring 50x, fixed 1.340 V, LLC Level 1. Validated with 3 x 30 minutes of Cinebench 2026 plus GPU heat, zero errors.
- **RAM: in progress.** A stable base is found (tRFC 680, tWR 24, tRCD/tRP 50, tCL 40, tRAS 96: about 74 ns from 75.2 ns at XMP). The voltage pass is running. A RAM water block is planned, and timings that depend on DIMM temperature wait for it. Final results will be added here.

## Licence

The text in this repository is licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). See [LICENSE.md](LICENSE.md).
Linked sources belong to their authors.
