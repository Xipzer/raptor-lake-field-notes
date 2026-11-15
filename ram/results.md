# RAM results

Projector, Kingston FURY Renegade 2x24 GB DDR5-8400, SK hynix 24 Gb M-die. CPU at 57x / 1.340 V throughout.
Latency is our own pointer-chase figure (lower is better). It is not AIDA64 and does not compare with AIDA64 numbers.
DIMM temperatures are peaks during the test, cooler stick / hotter stick.

**Status: 15 November 2026, morning. The RAM tune is finished for now.**
The final set below passed a three-hour memory validation on 15 November. The DIMM water block may move tCL and tREFI later.

Test per step in the sweep: y-cruncher FFTv4 + N63 + VT3, 7 minutes, 30 GB, plus WHEA check. Since 14 November the long checks add TestMem5 and an at-rest test. See [method.md](method.md).

## Starting point (XMP Profile 1)

DDR5-8400 (8380 effective), Gear 2, CL40-52-52-132, tRFC/tRFC2 922, tRFCpb 796, tWR 48, tRTP 23, tRRD_L 12, tRRD_S 8, tFAW 32, tCWL 40,
tWRRD_dg/sg 62/94 (so tWTR_S/L 16/48), tRDWR_dg/sg 22/22, tRDRD_sg/dg 16/8, tWRWR_sg/dg 16/8, tREFI 32767.
DIMM VDD/VDDQ 1.45 V, VDD2 (VDD_CPU) 1.45 V, TX (VDD_IMC) 1.45 V, VCCSA 1.25 V.

## Final set

Validated 15 November 04:06 to 06:55 (Measured, DIMM fan on, no GPU heat, CPU load kept cool):

| Check | Result |
|---|---|
| At-rest test: 30 GiB filled, locked in memory, checked every 2 minutes for 45 minutes, y-cruncher in 8 GB beside it | 23 checks, 0 flips. DIMMs up to 48.0 / 53.0 C |
| y-cruncher VT3 / FFTv4 / N63, 60 minutes | 30 sections passed, WHEA 0. DIMMs up to 49.0 / 54.8 C |
| TestMem5 0.13.1, anta777 config, 60 minutes | "Testing completed", 81% of memory. DIMMs up to 54.2 / **62.0 C** |

| Setting | XMP | Final | Survived / failed | Note |
|---|---|---|---|---|
| tCL | 40 | **38** | 38 / 36 | 36 trains at 1.50 V but corrupts memory at boot. 38 needs DIMM 1.50 / 1.48 V. Waits for the water block |
| tCWL | 40 | **34** | 34 / not tried lower | Tried alone at tCL 38 on 15 Nov |
| tRCD | 52 | **49** | 49 / 48 | 48 (with tRP 48) crash-looped at startup twice |
| tRCDW | 52 | **32** | 32 / not tried lower | 44, 40, 36, 32 all passed |
| tRP | 52 | **46** | 46 / not tried lower | 46 was the floor of the ladder. 45 not tried |
| tRAS | 132 | **64** | | Below tRCD + tRTP it stops being the binding limit. No gain lower |
| tRTP | 23 | **12** (entered 13) | | 12 is the DRAM minimum. The BIOS shows entered minus 1 |
| tWR | 48 | **12** | 12 / not tried lower | |
| tRRD_L | 12 | **11** | 11 / 10 | 10 crash-looped at startup twice |
| tRFC / tRFC2 | 922 | **660** | 660 / 650 | 650 failed run 2 when warm. 630 did not train |
| tRFCpb | 796 | **420** | 420 / 400 | 400: blue screen (MEMORY_MANAGEMENT) at startup, then no train |
| tREFI | 32767 | **32767** | 32767 / 49151 | 65535 and 49151 fail TestMem5 with the hotter stick near 60 C. See below |
| tWRRD_dg | 62 | **46** | 46 / 44 | 44: no train |
| tWRRD_sg | 94 | **64** | 64 / 62 | 62: y-cruncher failed at once, with cool sticks |
| tWTR_S / tWTR_L | 16 / 48 | 6 / 24 (follows) | | Estimate from tWRRD - tCWL - 6. Not set directly |
| tRDWR_dg / tRDWR_sg | 22 / 22 | **19 / 19** | 19 / 18 | 18: no train, as a pair and each alone |
| tRDRD_sg | 16 (trained) | **15** | 15 / 14 | 14: no POST |
| tWRWR_sg | 16 (trained) | **15** | 15 / not tried lower | |
| Command Rate | 2N | 2N | 2N / 1N | 1N: no train |
| TX (VDD_IMC) | 1.45 V | **1.35 V** | | 1.30 V passed short runs. 1.35 V kept for training margin |
| VDD2 (VDD_CPU) | 1.45 V | 1.45 V | 1.45 / 1.42 V | 1.42 V: test error in 3 minutes |
| VCCSA | 1.25 V | 1.25 V | | Not moved. No speed in it at 8400 |
| DIMM VDD / VDDQ | 1.45 / 1.45 V | **1.50 / 1.48 V** | | Only for tCL 38. Delivery verified: 1.500 / 1.485 V |
| CPU E-cores | 45x | **46x** | 46 / 47 | 47: hard freeze. See [../cpu/results.md](../cpu/results.md) |
| CPU ring | 50x | 50x | 50 / 51 | 51 fails Cinebench with GPU heat at 1.340, 1.350 and 1.360 V |

Ruled out on purpose: 8600 (timings are fixed in nanoseconds, so tCL would likely round up to 40 and latency would not improve),
lower frequency with tighter timings (same nanoseconds, less bandwidth). MRC Fast Boot stays off: full training at every start.

## The heat-bound fault (tREFI)

This was the main finding of 14 November. The facts, in order (all Measured):

| tREFI | tRFC / tRFCpb | Test | Hotter stick | Result |
|---|---|---|---|---|
| 65535 | 660 / 420 | at-rest 45 min, y-cruncher 60 min | up to 52 C | PASS, both |
| 65535 | 660 / 420 | TestMem5, DIMM fan unplugged | 60.2 C | "Error in test 0" at 50 min |
| 65535 | 688 / 460 | TestMem5 | 58.8 C | "An error 510173 occurred in the program" at 9 min |
| 65535 | 688 / 460 | TestMem5, fan on, GPU heat on | 61.5 C | "Error in test 0" at 16 min |
| 32767 | 688 / 460 | TestMem5, fan on, no GPU heat | 59.8 C | PASS, 60 min |
| 32767 | 660 / 460 | TestMem5 | 58.0 C | PASS, 60 min |
| 32767 | 660 / 420 | TestMem5 | 61.5 C | PASS, 60 min |
| 49151 | 660 / 420 | TestMem5 | 61.5 C | "An error -5837 occurred in the program" at 14 min |
| 32767 | final set | TestMem5 | 62.0 C | PASS, 60 min |

What it shows:

- **Looser refresh times did not fix it. The refresh interval did.** tRFC 688 / tRFCpb 460 still failed at 65535. tREFI 32767 passed at the same heat, and then tRFC 660 and tRFCpb 420 passed again on top of it.
- **Short runs and y-cruncher missed it.** tREFI 65535 had passed about 60 short y-cruncher runs and both 60-minute runs. TestMem5 at about 60 C on the hotter stick found it in 16 to 50 minutes.
- **The two sticks differ by 6 to 9 C.** The hotter one sets the limit.
- Earlier on 14 November tREFI 98304 and 131071 had looked fine in short runs. 131071 then blue-screened (BAD_POOL_CALLER) in its second run. Both are above 32767 and out.

What it means for daily use (Estimate): gaming barely loads the RAM. Running a large AI model from system RAM is the realistic hot case.
A DIMM water block is coming. After it, tREFI above 32767 and tCL 36 get a re-run.

## What it measures

| | XMP | Stable base of plan v1 | Final set |
|---|---|---|---|
| Latency, ns | 75.2 | 74.1 to 74.5 | about 70.5 to 71 |
| Read bandwidth, GB/s | 127.9 | 125 to 127 | about 124 to 130 |
| Cinebench 2026 multi, 10 min with GPU heat | 9,885 (E 45x) | | 10,059.55 (E 46x, 13 Nov set) |
| Cinebench 2026 multi, 30 min with GPU heat | 9,875 to 9,880 (E 45x) | | 10,028.45 (E 46x, 14 Nov set) |

Latency is our own pointer-chase figure. It is not AIDA64. Bandwidth moves 2 to 4 GB/s between runs, so read that row as "no worse".
All sweep figures were taken with a local model server holding most of the GPU memory and 5 to 9 GB of system RAM. Step-to-step comparisons stay fair.
The before-and-after benchmarks with every background load off are still owed.

Where the latency came from, roughly (run pairs before and after each change):

| Change | Latency moved by about |
|---|---|
| tRFC 922 to 680, tWR 48 to 24, tRCD/tRP 52 to 50, tRAS 132 to 96 | 1 ns |
| TX down, tWRRD, tRDWR | nothing we could see |
| tRFCpb 796 to 560 | 1.5 ns |
| tRFCpb 560 to 420, tRAS 96 to 76, tCWL 40 to 38 | 0.5 ns |
| tREFI 32767 to 65535 | 0.5 ns |
| tCL 40 to 38 with tCWL 36 | 0.5 ns |
| tRFC 680 to 660, tRCD/tRP 49, tRRD_L 11, tWR 20 | nothing we could see |
| Everything after 13 Nov evening (tRTP, tWR 12, tRP 46, tRCDW, tCWL 34, tRDRD_sg / tWRWR_sg 15), and tREFI back to 32767 | nothing we could see |

Run-to-run spread is about 0.5 to 1 ns, so read this table as a sketch.
Giving tREFI back from 65535 to 32767 cost no latency we could measure. The cost is in refresh time (about 2.0% against 1.0%, Estimate), not in the pointer chase.
The tRFCpb step is the surprise: research doubted the board even uses that timing. The drop from about 74.5 to about 72.6 ns suggests it does.
One caution: that step came right after the no-POST recovery and a full retrain, so a different training result may be part of the drop.

## Every step, in order

Outcome types are defined in [method.md](method.md): PASS, FAIL (test), CRASH, NO TRAIN, NOT APPLIED, NO POST, FREEZE.

### Plan v1 (12 to 13 November, one run per step, XMP voltages)

| # | Step | Outcome | Latency ns | Bandwidth GB/s | DIMM peak C | Notes |
|---|---|---|---|---|---|---|
| 0 | XMP baseline | PASS | 75.2 | 127.9 | 53 to 57 | First logged as FAIL by a tooling bug |
| 1 | tRFC 820 | PASS | 74.5 | 125.8 | not recorded | DIMM sensors read 0 after a cold boot |
| 2 | tRFC 740 | (write skipped) | | | | SCEWIN ignored the question: script built from a stale export |
| 3 | tRFC 740 | PASS | 74.7 | 128.3 | 62.8 / 64.8 | |
| 4 | tRFC 680 | PASS | 74.7 | 127.9 | 63.5 / 65.5 | |
| 5 | tRFC 630 | NO TRAIN | | | | Boot 282 s, BIOS ended at tRFC 395 |
| 6 | tWR 40 + tRTP 20 | NOT APPLIED | | | | tRTP read back 19. We misread this as "tRTP follows tWR". It does not: see [quirks.md](quirks.md) |
| 7 | tWR 40 | PASS | 75.0 | 126.1 | 61.0 / 62.5 | |
| 8 | tWR 36 | PASS | 74.7 | 126.7 | 63.2 / 65.2 | |
| 9 | tWR 30 | PASS | 74.8 | 125.2 | 63.5 / 65.5 | |
| 10 | tWR 24 | PASS | 75.0 | 125.5 | 63.5 / 65.8 | |
| 11 | tRRD_L 10 | CRASH | | | | Bugcheck 0x80 NMI at boot, frozen for about an hour until a manual reset. Next boot: 0xA at the Welcome screen. The BIOS had reverted the pending value to Auto by itself |
| 12 | tWTR_L 36, tWTR_S 12 | NOT APPLIED | | | | Read back 42 / 18, boot 44 s. tWTR is set through tWRRD |
| 13 | tRCD/tRP 50 | PASS | 74.3 | 125.9 | 60.8 / 62.8 | |
| 14 | tRCD/tRP 48 | CRASH | | | | Repeated crashes during startup, then Windows Recovery. No bugcheck code (dumps off), no WHEA |
| | failsafe test | | | | | Armed with unchanged settings. Fired at boot 2 and restored as designed |
| 15 | tCL 38 | PASS (once) | 73.6 | 126.8 | 45.2 / 50.5 | First step with the DIMM fan, from a cold start. Later shown marginal |
| 16 | tCL 36 | NO TRAIN | | | | Boot 135 s. Fallback tRFC 395, tWR 54, tCL/tRCD/tRP 29. Recovered unattended |
| 17 | tRAS 96 | PASS | 73.4 | 125.0 | 49.5 / 54.2 | |
| 18 | tRAS 76 | PASS | 73.5 | 125.8 | 49.8 / 54.8 | |
| 19 | tRAS 64 | FAIL | 73.4 | 125.8 | 50.8 / 55.5 | 6 of 7 sections. On a marginal base. Passed later (step 72) |
| 20 | tREFI 65535 | FAIL | 71.7 | 124.2 | 49.2 / 53.8 | 3 of 7 sections. On a marginal base. Passed later (step 56) |
| 21 | tRFC 670 | FAIL | 73.2 | 126.4 | 50.5 / 55.2 | 6 of 7 sections. On a marginal base. Passed later (step 59) |
| 22 | Control A: kept settings (tCL 38, tRAS 76), no change | FAIL | 73.4 | 125.3 | 49.2 / 53.2 | The base itself is marginal. Steps 19 to 21 are not proven fails |
| 23 | Control A, run 2 | (did not run) | | | | Script tripped on a locked log file |
| 24 | Control B: tRAS back to 96 | FAIL | 73.2 | 125.5 | 50.0 / 54.5 | FFTv4 checksum mismatch at 6:18, reported on an E-core. An in-cache y-cruncher run passed 8 min, so the error needs memory traffic |
| 25 | Control C: tCL back to 40, tRAS 96 | PASS | 74.1 | 126.8 | 49.8 / 54.8 | |
| 26 | Control C, run 2 | PASS | 74.5 | 125.2 | 50.2 / 55.8 | **Stable base found** |

Stable base after step 26: tRFC/tRFC2 680, tWR 24, tRCD/tRP 50, tCL 40, tRAS 96, everything else XMP.

### Plan v2 (13 November, two runs per step)

Latency and bandwidth are run 1 / run 2. DIMM peak is from run 2, which is the warmer one.

| # | Step | Outcome | Latency ns | Bandwidth GB/s | DIMM peak C | Notes |
|---|---|---|---|---|---|---|
| 27 | TX 1.400 V | PASS | 73.6 / 74.5 | 125.7 / 123.3 | 50.5 / 55.8 | |
| 28 | TX 1.350 V | PASS | 74.4 / 74.3 | 123.0 / 126.5 | 50.8 / 56.0 | |
| 29 | TX 1.300 V | PASS | 73.9 / 74.5 | 125.4 / 125.2 | 50.5 / 56.0 | |
| 30 | VDD2 1.420 V | FAIL | 73.7 | 124.6 | 43.8 / 47.8 | 1 of 7 sections passed, then ConvolutionFailedException, within 3 minutes. VDD2 stays 1.45 V |
| 31 | tWRRD_dg 56 | PASS | 73.9 / 75.3 | 124.5 / 126.0 | 51.5 / 56.8 | |
| 32 | tWRRD_dg 52 | PASS | 73.8 / 74.4 | 125.6 / 125.5 | 51.8 / 57.0 | |
| 33 | tWRRD_dg 48 | PASS | 74.5 / 74.9 | 124.9 / 125.7 | 52.0 / 57.5 | |
| 34 | tWRRD_sg 80 | PASS | 74.0 / 74.4 | 125.0 / 125.5 | 52.0 / 57.2 | |
| 35 | tWRRD_sg 70 | PASS | 73.7 / 74.8 | 126.6 / 125.2 | 51.8 / 57.5 | |
| 36 | tWRRD_sg 64 | NO TRAIN | | | | Boot 153 s. Fallback: tRFC 395, tWR 54, tCL/tRCD/tRP 29, tRAS 57, TX 1.100 V. Recovered unattended. Passed on 13 and 14 Nov, on a different base |
| 37 | tRDWR_dg 20 | PASS | 74.2 / 74.9 | 126.7 / 124.7 | 52.0 / 57.2 | |
| 38 | tRDWR_dg 19 | PASS | 74.1 / 74.6 | 126.1 / 125.7 | 52.0 / 57.2 | |
| 39 | tRDWR_sg 20 | PASS | 73.8 / 74.7 | 124.4 / 126.7 | 52.0 / 57.2 | |
| 40 | tRDWR_sg 19 | PASS | 73.9 / 74.7 | 124.5 / 125.2 | 52.0 / 57.2 | |
| 41 | tRDRD_sg 15 | PASS | 73.8 / 74.5 | 126.7 / 125.5 | 52.0 / 57.2 | Reverted after step 42, then re-tested and kept on 14 Nov |
| 42 | tRDRD_sg 14 | **NO POST** | | | | No picture. Five manual power cycles. Memory came up at DDR5-3600 fail-safe with NVRAM intact. See [method.md](method.md) section 8 |
| | plan change | | | | | tRDRD_sg back to trained 16, tWRWR_sg dropped for then, TX 1.30 to 1.35 V for training margin |
| 43 | tRFCpb 560 | PASS | 72.5 / 72.8 | 125.3 / 124.0 | 51.2 / 56.5 | |
| 44 | tRFCpb 460 | PASS | 76.6 / 72.0 | 124.5 / 127.5 | 51.8 / 57.0 | The 76.6 is an outlier reading |
| 45 | tRAS 84 | PASS | 71.8 / 72.3 | 126.9 / 126.9 | no readings | |
| 46 | tRAS 76 | PASS | 71.8 / 72.4 | 124.6 / 125.8 | 52.0 / 57.2 | |
| 47 | tCWL 38 (at tCL 40) | PASS | 72.1 / 72.2 | 125.7 / 127.8 | 52.2 / 57.5 | |
| 48 | tREFI 40960 | PASS | 71.3 / 71.9 | 123.0 / 125.9 | 52.5 / 57.8 | |
| 49 | tREFI 49152 | PASS | 71.4 / 72.0 | 124.4 / 126.7 | 52.0 / 57.8 | |
| 50 | CPU ring 51x | FAIL | 71.1 / 71.7 | 125.7 / 127.4 | 51.2 / 56.8 | Both memory runs passed. Cinebench with GPU heat crashed at 2:24 |
| 51 | CPU E-cores 46x | PASS | 71.8 / 72.1 | 127.6 / 126.2 | 51.8 / 57.0 | Cinebench 10 min with GPU heat: 10,059.55 |
| | "discovery done" | | | | | Final validation started and was stopped 4 minutes in: the owner asked for an extras pass first |
| 52 | tRFCpb 440 | PASS | 71.6 / 71.9 | 126.2 / 125.7 | 52.0 / 57.5 | |
| 53 | tRFCpb 420 | PASS | 71.2 / 71.5 | 125.0 / 126.8 | 52.0 / 57.2 | |
| 54 | tREFI 57344 | PASS | 71.1 / 71.6 | 128.3 / 126.0 | 52.2 / 57.5 | |
| 55 | tREFI 65535 | PASS | 70.9 / 71.5 | 125.3 / 125.9 | 52.0 / 57.5 | |
| 56 | tCL 38 + tCWL 36, DIMM 1.45 V | FAIL | 70.3 | 126.4 | 50.2 / 54.8 | 6 of 7 sections. Same late error as the controls: tCL 38 at 1.45 V is marginal |
| 57 | tCL 38 + tCWL 36, DIMM VDD/VDDQ 1.50/1.48 V | PASS | 70.3 / 71.1 | 126.7 / 127.8 | 53.0 / 58.8 | |
| 58 | tRFC 670 | PASS | 70.3 / 70.8 | 127.5 / 126.7 | 53.2 / 59.0 | |
| 59 | tRFC 660 | PASS | 70.1 / 70.8 | 124.8 / 126.7 | 52.8 / 58.8 | |
| | E-cores 47x, first try | (nothing written) | | | | Tooling fault: the BIOS read raced its helper |
| 60 | CPU E-cores 47x | **FREEZE** | | | | Hard freeze about 3 minutes into run 1. Power button. First logged as CRASH |
| | ring 51x + Vcore 1.350 V, first try | (not tested) | | | | Logged as CRASH by mistake: the crash check picked up the freeze before it |
| 61 | CPU ring 51x + Vcore 1.350 V | FAIL | 70.3 / 70.6 | 125.6 / 126.2 | 53.0 / 58.8 | Both memory runs passed. Cinebench with GPU heat crashed at 4:13, P7 at 106 C |
| 62 | tRCD/tRP 49 | PASS | 70.6 / 71.0 | 127.6 / 129.2 | 53.8 / 59.2 | The boot before this test took about 7 minutes instead of 2. Most likely a 0x9F shutdown hang, not training (see [quirks.md](quirks.md)) |
| 63 | tRRD_L 11 | PASS | 70.5 / 71.2 | 130.4 / 129.5 | 53.5 / 59.2 | |
| 64 | tWR 22 | PASS | 70.5 / 71.1 | 129.0 / 129.1 | 53.0 / 58.8 | |
| 65 | tWR 20 | PASS | 70.5 / 71.1 | 127.8 / 127.9 | 52.8 / 58.8 | tRTP still reads 23 with tWR at 20 |
| 66 | tRTP 19 | NOT APPLIED | | | | The BIOS read back 18, boot 42 s. No test ran |
| 67 | tRAS 72 | PASS | 70.4 / 70.8 | 130.4 / 128.0 | 52.8 / 58.2 | |
| 68 | tRAS 68 | PASS | 70.4 / 71.2 | 127.7 / 128.3 | 52.8 / 58.5 | Below tRCD + tRTP = 72, the "floor" we expected it to fail at |
| 69 | tRTP 20 entered (reads 19) | PASS | 70.4 / 71.0 | 126.0 / 129.1 | 53.0 / 58.8 | Checker now expects entered minus 1 |
| 70 | tRTP 16 entered (reads 15) | PASS | 70.5 / 71.1 | 126.3 / 127.1 | 52.8 / 58.5 | |
| 71 | tRTP 13 entered (reads 12) | PASS | 70.3 / 71.1 | 127.7 / 130.2 | 53.0 / 58.5 | DRAM minimum |
| 72 | tRAS 64 | PASS | 70.6 / 70.9 | 129.9 / 129.9 | 52.8 / 58.5 | |
| 73 | tRCD/tRP 48 (retry) | CRASH | | | | Startup crash loop again, three blue screens. The failsafe never ran. The owner set 49 in the BIOS |
| 74 | tRRD_L 10 (retry) | CRASH | | | | Startup crash loop. This time the failsafe recovered it unattended |

Count for plan v2 (steps 27 to 74): 38 PASS, 4 FAIL (two of them the ring), 2 CRASH, 1 NO TRAIN, 1 NOT APPLIED, 1 NO POST, 1 FREEZE.

### Coverage audit and the night of 13 to 14 November

After the first "discovery done" a coverage audit listed every setting not yet tested to its limit. The queue ran one step per boot.
Since 13 November 23:00 a step that fails is always restored, rebooted and confirmed by export before the next one is written.

| Step | Outcome | Notes |
|---|---|---|
| tWR 18, 16 | PASS | |
| tWRRD_dg 46 | PASS | |
| tWRRD_dg 44 | NO TRAIN | Recovered unattended |
| tWRRD_sg 68, 66, 64 | PASS | Re-tested later on a clean base: pass again |
| tWRRD_sg 62 | FAIL | y-cruncher failed at once, DIMMs 35 / 37 C. Not heat |
| tCL 36 + tCWL 34 at DIMM 1.50 / 1.48 V | **UNSTABLE BOOT** | Trained and booted, but corrupted memory: boot files failed their checksum while loading, Windows Recovery twice, a blue screen. The files on disk were intact. The owner set all timings to Auto in the BIOS |
| tRP 48 alone | PASS | On a clean re-test. So of the 48 / 48 pair, tRCD is the weak one |
| tRDWR_dg + _sg 18 | NO TRAIN | Recovered unattended. Later each field alone: NO TRAIN too |
| Command Rate 1N | NO TRAIN | Recovered unattended |
| tRCDW 44, 40 | PASS | |
| tRDRD_sg 15 | PASS | Kept this time. Its slow first boot was a shutdown hang, not training |
| Ring 51x at 1.360 V | FAIL | Cinebench with GPU heat crashed at 3:35, P7 105 C, 428 W |
| tREFI 98304 | PASS | Two runs, twice |
| tREFI 131071 | CRASH | BAD_POOL_CALLER blue screen 10 to 25 s into run 2, DIMMs 52 / 57 C |
| tRFC 650 | FAIL | Run 2: test error at 52 / 58 C. A control of the base passed |
| tRFCpb 400 | CRASH | MEMORY_MANAGEMENT blue screen at startup, then two fallback boots |

### 14 November: first long validations

| What | Outcome | Notes |
|---|---|---|
| y-cruncher 60 min, tREFI 98304 | CRASH | Reset about 25 minutes in, no WHEA record, no stop code (dumps off). Our script restarted the 60 minutes blind. tREFI given back to 65535 |
| y-cruncher 60 min, tREFI 65535 | PASS | 30 sections |
| Cinebench 30 min with GPU heat | PASS | 10,028.45, 421 W, P7 106 C, water to 36.6 C |
| Five hot reboots | PASS | Boots 19, 323, 21, 20, 21 s. The 323 s one was a 0x9F shutdown hang |
| At-rest test with y-cruncher and GPU heat beside it | **hard reset** | 5 minutes in, 459 W, P6 107 C. Two clean memory checks before it. No stop code with dumps on: a CPU or platform fault under heat, not memory. Our error: GPU heat on top of an all-thread memory test made it the heaviest CPU load of the whole campaign |

After the hard reset the owner set the rules for every later test: no GPU heat, the CPU load kept cool, and a stop if the hottest core reaches 100 C.

### 14 to 15 November: the memory track

Memory tests with the CPU kept cool (y-cruncher confined to the E-cores and the cooler P-cores, later all cores once the repaste brought temperatures down), no GPU heat, no reboots inside a check.

| Step | Outcome | Notes |
|---|---|---|
| At-rest 45 min, y-cruncher 60 min (tREFI 65535) | PASS | Hotter stick up to 52 C |
| TestMem5 60 min (tREFI 65535) | FAIL | See the tREFI table above. Then tRFC 688 / tRFCpb 460, still FAIL |
| (CPU repaste, 14 Nov 21:00 to 22:12) | | Planned power-off |
| tREFI 32767, then tRFC 660, then tRFCpb 420 | PASS each | TestMem5 60 min each |
| tREFI 49151 | FAIL | TestMem5 program error at 14 min, hotter stick 61.5 C |
| tRP 47, 46 | PASS | One y-cruncher run each |
| tRCDW 36, 32 | PASS | |
| tWR 14, 12 | PASS | |
| tCWL 34 alone | PASS | |
| tWRWR_sg 15 | PASS | |
| Final validation | **PASS** | At-rest 45 min, y-cruncher 60 min, TestMem5 60 min. See "Final set" |

## DIMM temperature

| Condition | Steady peak, cooler / hotter stick |
|---|---|
| No DIMM fan, DIMM 1.45 V (steps 3 to 14) | 60.8 to 63.5 / 62.5 to 65.8 C |
| DIMM fan, DIMM 1.45 V (steps 17 to 55) | 49.5 to 52.5 / 54 to 57.8 C |
| DIMM fan, DIMM 1.50 V, y-cruncher | 50 to 54 / 55 to 59 C |
| DIMM fan, DIMM 1.50 V, TestMem5 | 54 to 55 / 58 to 62 C |
| DIMM fan, DIMM 1.50 V, at-rest test with y-cruncher in 8 GB | 45 to 48 / 49 to 53 C |

The fan is worth 11 to 12 C on the cooler stick and 8 to 9 C on the hotter one. The two sticks run 6 to 9 C apart.
TestMem5 heats the sticks about 3 C more than y-cruncher. Raising DIMM VDD/VDDQ from 1.45 to 1.50/1.48 V cost about 1 to 1.5 C.

Public guidance puts the comfort zone for 24 Gb M-die under about 45 to 50 C, and says tREFI 65535 errors from about 47 C.
Ours passed tREFI 65535 in every short run at up to 59 C, and failed TestMem5 at about 60 C. In the end the public guidance was closer to the truth than our short runs.

## Old failures, re-read

| Old result | What we thought | What we know now |
|---|---|---|
| tCL 38 PASS (step 15) | tCL 38 works | Marginal at 1.45 V. Needs DIMM 1.50/1.48 V (steps 56, 57) |
| tRAS 64 FAIL | the tRAS floor | The marginal base. tRAS 68 and 64 passed later, below tRCD + tRTP |
| tREFI 65535 PASS (step 55 on) | heat is fine at 57.5 C | Fine for 7 minutes. Fails TestMem5 near 60 C in 16 to 50 minutes |
| tRCD/tRP 48 CRASH, tRRD_L 10 CRASH | maybe heat, maybe the two-clock jump | Both crash-looped again on the single-step retry. tRP 48 alone passes, so tRCD is the weak one |
| tWTR 36/12 NOT APPLIED | "the BIOS will not go below 42/18" | Wrong. tWTR follows tWRRD and tCWL |
| Slow boots of 323 to 353 s | slow memory training when hot | Windows shutdown hangs (bugcheck 0x9F) in the graphics driver. See [quirks.md](quirks.md) |

## Still to come

- Before-and-after benchmarks against the original profile, with every background load off.
- After the DIMM water block: tCL 36, tREFI above 32767, tRFC below 660. Then a long re-validation.
- Positive controls for the at-rest test (run it on a tREFI known to fail, to see whether it catches the fault) were planned and not yet run.
- A true cold start (power off for minutes) has not been tested.
