# RAM results

Projector, Kingston FURY Renegade 2x24 GB DDR5-8400, SK hynix 24 Gb M-die. CPU at 57x / 1.340 V throughout.
Test per step: y-cruncher FFTv4 + N63 + VT3, 7 minutes, 30 GB, plus WHEA check. See [method.md](method.md).
Latency is our own pointer-chase figure (lower is better). It is not AIDA64 and does not compare with AIDA64 numbers.
DIMM temperatures are peaks during the test, cooler stick / hotter stick.

**Status: 13 November 2026, late afternoon. The sweep is still running.**
Everything below is from the journal up to 17:00. The kept set can still change.
No kept value has had a long validation run yet. "PASS" in plan v2 means two clean 7-minute runs in one boot, nothing more.

## Starting point (XMP Profile 1)

DDR5-8400 (8380 effective), Gear 2, CL40-52-52-132, tRFC/tRFC2 922, tRFCpb 796, tWR 48, tRTP 23, tRRD_L 12, tRRD_S 8, tFAW 32, tCWL 40,
tWRRD_dg/sg 62/94 (so tWTR_S/L 16/48), tRDWR_dg/sg 22/22, tRDRD_sg/dg 16/8, tWRWR_sg/dg 16/8, tREFI 32767.
DIMM VDD/VDDQ 1.45 V, VDD2 (VDD_CPU) 1.45 V, TX (VDD_IMC) 1.45 V, VCCSA 1.25 V.

## Kept set so far

| Setting | XMP | Kept | First value that did not make it | Note |
|---|---|---|---|---|
| tCL | 40 | **38** | 36: NO TRAIN | 38 needs DIMM VDD/VDDQ 1.50/1.48 V. At 1.45 V it is marginal |
| tCWL | 40 | **36** | | 38 passed at tCL 40. 36 went in together with tCL 38 |
| tRCD / tRP | 52 | **49** | 48: CRASH (on 12 Nov, from 50, no DIMM fan) | 48 is queued for a retry |
| tRAS | 132 | **76** | | 72 is under test (run 1 of 2 passed). Floor is tRCD + tRTP = 49 + 23 = 72. The old 64 was below the floor |
| tRFC / tRFC2 | 922 | **660** | 630: NO TRAIN | 660 is 157 ns |
| tRFCpb | 796 | **420** | | 420 is 100 ns. Nothing lower tried |
| tREFI | 32767 | **65535** | | Passed at DIMMs 52 / 57.5 C |
| tWR | 48 | **20** | | 40, 36, 30, 24, 22, 20 all passed. tRTP still reads 23 |
| tRTP | 23 | 23 | 19: NOT APPLIED | The BIOS read back 18. Untuned |
| tRRD_L | 12 | **11** | 10: CRASH (on 12 Nov, from 12, no DIMM fan) | 10 is queued for a retry |
| tWRRD_dg | 62 | **48** | | Nothing lower tried |
| tWRRD_sg | 94 | **70** | 64: NO TRAIN | |
| tWTR_S / tWTR_L | 16 / 48 | 6 / 28 (follows) | | Not set directly. It is tWRRD minus tCWL minus 6 on this board |
| tRDWR_dg / tRDWR_sg | 22 / 22 | **19 / 19** | | Nothing lower tried |
| tRDRD_sg | 16 | 16 (trained) | 14: NO POST | 15 passed, then we went back to trained on purpose |
| tWRWR_sg | 16 | 16 (trained) | | Dropped from the plan after the no-POST |
| TX (VDD_IMC) | 1.45 V | **1.35 V** | | 1.30 V passed too. Raised back to 1.35 V for training margin |
| VDD2 (VDD_CPU) | 1.45 V | 1.45 V | 1.42 V: FAIL | Failed within 3 minutes, after one test section |
| VCCSA | 1.25 V | 1.25 V | | Not touched |
| DIMM VDD / VDDQ | 1.45 / 1.45 V | **1.50 / 1.48 V** | | Only for tCL 38. Delivery verified: 1.500 / 1.485 V |
| CPU E-cores | 45x | **46x** | 47x: FREEZE | See [../cpu/results.md](../cpu/results.md) |
| CPU ring | 50x | 50x | 51x: FAIL | Fails Cinebench with GPU heat at 1.340 and 1.350 V |

Side effects we checked against the XMP export and found expected:
tRC 276 to 189, tXSR to 660, RTL down 2 (tCL 38), tWRPRE and tWRPDEN follow tWR, tRCDW follows tRCD.

## What it measures

| | XMP | Stable base of plan v1 | Kept set now |
|---|---|---|---|
| Latency, ns | 75.2 | 74.1 to 74.5 | 70.5 to 71.2 |
| Read bandwidth, GB/s | 127.9 | 125 to 127 | 127.8 to 130.4 |
| Cinebench 2026 multi, 10 min with GPU heat | 9,885 (E 45x) | | 10,059.55 (E 46x) |

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

Run-to-run spread is about 0.5 to 1 ns, so read this table as a sketch.
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
| 19 | tRAS 64 | FAIL | 73.4 | 125.8 | 50.8 / 55.5 | 6 of 7 sections. On a marginal base, and below the tRAS floor of 73 |
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
| 36 | tWRRD_sg 64 | NO TRAIN | | | | Boot 153 s. Fallback: tRFC 395, tWR 54, tCL/tRCD/tRP 29, tRAS 57, TX 1.100 V. Recovered unattended |
| 37 | tRDWR_dg 20 | PASS | 74.2 / 74.9 | 126.7 / 124.7 | 52.0 / 57.2 | |
| 38 | tRDWR_dg 19 | PASS | 74.1 / 74.6 | 126.1 / 125.7 | 52.0 / 57.2 | |
| 39 | tRDWR_sg 20 | PASS | 73.8 / 74.7 | 124.4 / 126.7 | 52.0 / 57.2 | |
| 40 | tRDWR_sg 19 | PASS | 73.9 / 74.7 | 124.5 / 125.2 | 52.0 / 57.2 | |
| 41 | tRDRD_sg 15 | PASS | 73.8 / 74.5 | 126.7 / 125.5 | 52.0 / 57.2 | Not kept in the end (back to trained 16) |
| 42 | tRDRD_sg 14 | **NO POST** | | | | No picture. Five manual power cycles. Memory came up at DDR5-3600 fail-safe with NVRAM intact. See [method.md](method.md) section 8 |
| | plan change | | | | | tRDRD_sg back to trained 16, tWRWR_sg dropped, TX 1.30 to 1.35 V for training margin |
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
| 62 | tRCD/tRP 49 | PASS | 70.6 / 71.0 | 127.6 / 129.2 | 53.8 / 59.2 | The boot before this test took about 7 minutes instead of 2. Cause not recorded |
| 63 | tRRD_L 11 | PASS | 70.5 / 71.2 | 130.4 / 129.5 | 53.5 / 59.2 | |
| 64 | tWR 22 | PASS | 70.5 / 71.1 | 129.0 / 129.1 | 53.0 / 58.8 | |
| 65 | tWR 20 | PASS | 70.5 / 71.1 | 127.8 / 127.9 | 52.8 / 58.8 | tRTP still reads 23 with tWR at 20 |
| 66 | tRTP 19 | NOT APPLIED | | | | The BIOS read back 18, boot 42 s. No test ran |
| 67 | tRAS 72 | under test | 70.4 / | 130.4 / | 52.0 / 57.2 (run 1) | Run 1 passed, run 2 was under way at 17:00. Exactly at the floor: tRCD 49 + tRTP 23 |

Count so far for plan v2 (steps 27 to 66): 32 PASS, 4 FAIL (two of them the ring), 1 NO TRAIN, 1 NOT APPLIED, 1 NO POST, 1 FREEZE.

## DIMM temperature

| Condition | Steady peak, cooler / hotter stick |
|---|---|
| No DIMM fan, DIMM 1.45 V (steps 3 to 14) | 60.8 to 63.5 / 62.5 to 65.8 C |
| DIMM fan, DIMM 1.45 V (steps 17 to 55) | 49.5 to 52.5 / 54 to 57.8 C |
| DIMM fan, DIMM 1.50 V (steps 57 on) | 52.8 to 53.8 / 58.8 to 59.2 C |

The fan is worth 11 to 12 C on the cooler stick and 8 to 9 C on the hotter one.
The 45.2 / 50.5 C at step 15 was a cold-start reading and is not a fair comparison.

Raising DIMM VDD/VDDQ from 1.45 to 1.50/1.48 V cost about 1 to 1.5 C.

Public guidance puts the comfort zone for 24 Gb M-die under about 45 to 50 C, and says tREFI 65535 errors from about 47 C.
Ours passed tREFI 65535 twice at 52 / 57.5 C. The seven steps kept after it also passed with tREFI 65535 in place, at up to 59 C.
That is two 7-minute runs per step, not a daily-use proof. The long validation and a summer day will have the last word.

## Old failures, re-read

| Old result | What we thought | What we know now |
|---|---|---|
| tCL 38 PASS (step 15) | tCL 38 works | Marginal at 1.45 V. Needs DIMM 1.50/1.48 V (steps 56, 57) |
| tRAS 64 FAIL | maybe the marginal base | Below the floor of tRCD + tRTP = 73. It would gain nothing even if it passed |
| tREFI 65535 FAIL, tRFC 670 FAIL | maybe heat | The marginal base. Both pass on a stable one, at the same temperatures |
| tRCD/tRP 48 CRASH, tRRD_L 10 CRASH | configuration failures, heat unproven | 49 and 11 pass with the fan. 48 and 10 are queued for a retry, one clock at a time |
| tWTR 36/12 NOT APPLIED | "the BIOS will not go below 42/18" | Wrong. tWTR follows tWRRD and tCWL and now reads 28 / 6 |

## Still to come

- tRTP: still 23. The one attempt read back one clock lower than entered and was not tested.
- tRAS 72 (under test, run 1 of 2 passed). 68 is below the floor unless tRTP comes down.
- Retries of tRCD/tRP 48 and tRRD_L 10.
- Final validation: y-cruncher 60 minutes, Cinebench 30 minutes with GPU heat, three retrain boots.
- Before and after benchmarks against the original profile.
- After the DIMM water block: a short re-run of the heat-limited items (tREFI, tCL, tRFC, DIMM voltage).
