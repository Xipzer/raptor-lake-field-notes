# RAM results

Projector, Kingston FURY Renegade 2x24 GB DDR5-8400, SK hynix 24 Gb M-die. CPU at 57x / 1.340 V throughout.
Test per step: y-cruncher FFTv4 + N63 + VT3, 7 minutes, 30 GB, plus WHEA check. See [method.md](method.md).
Latency is our pointer-chase figure (lower is better). DIMM temperatures are peaks during the test, DIMM A / DIMM B.

**Status: in progress.** Final results will be added after the voltage pass, the turnaround groups and the RAM water block.

## Starting point (XMP Profile 1)

DDR5-8400 (8380 effective), Gear 2, CL40-52-52-132, tRFC/tRFC2 922, tRFCpb 796, tWR 48, tRRD_L 12, tRRD_S 8, tFAW 32, tCWL 40, tWTR_L/S 48/16, tREFI 32767.
DIMM VDD/VDDQ 1.45 V, VDD2 (VDD_CPU) 1.45 V, TX (VDD_IMC) 1.45 V, VCCSA 1.25 V.

## Where it stands

| Timing | XMP | Kept | Failed at | Note |
|---|---|---|---|---|
| tRFC / tRFC2 | 922 | **680** | 630: NO TRAIN | 670 FAIL was on a marginal base, not proven |
| tWR | 48 | **24** | | 40, 36, 30, 24 all passed. The BIOS sets tRTP itself |
| tRRD_L | 12 | 12 | 10: CRASH | at boot, DIMMs cool |
| tWTR_L / tWTR_S | 48 / 16 | 48 / 16 | 36 / 12: NOT APPLIED | BIOS raised it to 42 / 18. Tune through tWRRD instead |
| tRCD / tRP | 52 | **50** | 48: CRASH | at boot, DIMMs cool |
| tCL | 40 | **40** | 38: marginal, 36: NO TRAIN | 38 passed once, then failed every control |
| tRAS | 132 | **96** | 76 in the marginal base | 64 FAIL not proven |
| tREFI | 32767 | 32767 | 65535: FAIL, not proven | waits for the water block |
| TX (VDD_IMC) | 1.450 V | **1.400 V** | | 1.350 V under test |

Latency on the stable base: **74.1 to 74.5 ns** (XMP: 75.2 ns). Bandwidth unchanged at about 125 to 128 GB/s.
The best single reading, 71.7 ns at tREFI 65535, came from a run that failed.

## Every step, in order

| # | Step | Outcome | Latency ns | Bandwidth GB/s | DIMM peak C | Notes |
|---|---|---|---|---|---|---|
| 0 | XMP baseline | PASS | 75.2 | 127.9 | 53 to 57 | First logged as FAIL by a tooling bug (see quirks) |
| 1 | tRFC 820 | PASS | 74.5 | 125.8 | not recorded | DIMM sensors read 0 after a cold boot |
| 2 | tRFC 740 | (write skipped) | | | | SCEWIN ignored the question: script built from a stale export |
| 3 | tRFC 740 | PASS | 74.7 | 128.3 | 62.8 / 64.8 | |
| 4 | tRFC 680 | PASS | 74.7 | 127.9 | 63.5 / 65.5 | |
| 5 | tRFC 630 | NO TRAIN | | | | Boot 282 s, BIOS ended at tRFC 395 |
| 6 | tWR 40 + tRTP 20 | NOT APPLIED | | | | BIOS derives tRTP from tWR (40 gives 19) |
| 7 | tWR 40 | PASS | 75.0 | 126.1 | 61.0 / 62.5 | |
| 8 | tWR 36 | PASS | 74.7 | 126.7 | 63.2 / 65.2 | |
| 9 | tWR 30 | PASS | 74.8 | 125.2 | 63.5 / 65.5 | |
| 10 | tWR 24 | PASS | 75.0 | 125.5 | 63.5 / 65.8 | |
| 11 | tRRD_L 10 | CRASH | | | | Bugcheck 0x80 NMI at boot, frozen for about an hour until a manual reset. Next boot: 0xA at the Welcome screen. The BIOS had reverted the pending value to Auto by itself |
| 12 | tWTR_L 36, tWTR_S 12 | NOT APPLIED | | | | BIOS set 42 / 18, boot 44 s |
| 13 | tRCD/tRP 50 | PASS | 74.3 | 125.9 | 60.8 / 62.8 | |
| 14 | tRCD/tRP 48 | CRASH | | | | Repeated crashes during startup, then Windows Recovery. No bugcheck code (dumps off), no WHEA |
| | failsafe test | | | | | Armed with unchanged settings. Fired at boot 2 and restored as designed |
| 15 | tCL 38 | PASS (once) | 73.6 | 126.8 | 45.2 / 50.5 | First step with the 120 mm DIMM fan. Later shown marginal |
| 16 | tCL 36 | NO TRAIN | | | | Boot 135 s. Fallback tRFC 395, tWR 54, tCL/tRCD/tRP 29. Recovered unattended |
| 17 | tRAS 96 | PASS | 73.4 | 125.0 | 49.5 / 54.2 | |
| 18 | tRAS 76 | PASS | 73.5 | 125.8 | 49.8 / 54.8 | |
| 19 | tRAS 64 | FAIL | 73.4 | 125.8 | 50.8 / 55.5 | 6 of 7 sections, AlgorithmFailedException |
| 20 | tREFI 65535 | FAIL | 71.7 | 124.2 | 49.2 / 53.8 | 3 of 7 sections |
| 21 | tRFC 670 | FAIL | 73.2 | 126.4 | 50.5 / 55.2 | 6 of 7 sections |
| 22 | Control A: kept settings (tCL 38, tRAS 76), no change | FAIL | 73.4 | 125.3 | 49.2 / 53.2 | The base itself is marginal. Steps 19 to 21 are not proven fails |
| 23 | Control A, run 2 | (did not run) | | | | Script tripped on a locked log file |
| 24 | Control B: tRAS back to 96 | FAIL | 73.2 | 125.5 | 50.0 / 54.5 | FFTv4 checksum mismatch at 6:18, reported on an E-core. An in-cache y-cruncher run passed 8 min, so the error needs memory traffic |
| 25 | Control C: tCL back to 40, tRAS 96 | PASS | 74.1 | 126.8 | 49.8 / 54.8 | |
| 26 | Control C, run 2 | PASS | 74.5 | 125.2 | 50.2 / 55.8 | **Stable base found** |
| 27 | TX 1.400 V, run 1 | PASS | 73.6 | 125.7 | 49.5 / 54.2 | Plan v2 starts. Two runs per step from here |
| 28 | TX 1.400 V, run 2 | PASS | 74.5 | 123.3 | 50.5 / 55.8 | Kept |
| 29 | TX 1.350 V | testing | | | | |

Stable base after step 26: tRFC/tRFC2 680, tWR 24, tRCD/tRP 50, tCL 40, tRAS 96, everything else XMP.

## DIMM temperature, before and after a fan

| Condition | Early in the test | Peak |
|---|---|---|
| No DIMM fan (steps 3 to 14) | 53 to 57 C | 60.8 to 65.8 C |
| 120 mm fan aimed at the DIMMs (step 15 on) | 38.8 / 42.5 C | 45.2 to 55.8 C |

Every pass in steps 3 to 13 was at 60 C or above. The research puts the comfort zone for 24 Gb M-die under about 45 to 50 C.
So the passes say the timings hold even hot. The failures at the hot end are not yet separated from heat.

tRRD_L 10 and tRCD/tRP 48 crashed during boot or login, before the DIMMs warmed up. We label them configuration failures, heat unproven.
They will be retried after the water block, one unit at a time.

## Queued

- Plan v2, now: TX 1.350 then 1.300 V; VDD2 1.420 then 1.400 V; tWRRD_dg 56..48; tWRRD_sg 84..64; tRDWR_dg 20, 19; tRDRD_sg; tWRWR_sg; tRFCpb 796 towards 420; tRAS 96 to 76.
- After the RAM water block (target DIMMs at or under 40 to 45 C): tCL 38 with tCWL 38, tRFC towards 651, tREFI, tRCD/tRP 49 then 48, tRRD_L 11 then 10.
- Final validation: VT3 60 minutes, TM5, cold-boot retrains, one run with GPU heat, benchmarks before and after.
