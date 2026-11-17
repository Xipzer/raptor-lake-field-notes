# CPU results

All runs on Projector. Load: Cinebench 2026.1.3 multi-core. E-cores 45x and ring 50x unless noted.
"GPU heat" means the RTX 5090 held at about 530 W in the same loop. Water is read at the pump (the coolest point).

## Result

**Final set (15 November, after the repaste): 57x all P-cores, E-cores 46x, ring 50x, fixed 1.320 V, LLC Level 1.**
In the BIOS, confirmed by a SCEWIN export.

- Proof for 57x / 1.320 V: two 30-minute runs of Cinebench with GPU heat, 0 errors. Scores 10,151.53 and 10,179.94.
  Hottest core P3 at 94 / 96 C, package about 401 W, water up to 39.9 / 40.1 C, WHEA 0.
- 1.310 V finished 30 minutes but logged 2 corrected errors on P2: FAIL. So 1.320 V is 10 mV above a failure in the hardest test we run. Borderline by design; 1.330 V also passed 30 minutes.
- Cost against 59x: 3.4% clock, 2.5% in Cinebench single-core. The original 59x profile was not stable anyway. See "Before and after" below.
- No corrected error has fallen inside a window on the final set. Every WHEA record on 15 November sits in a failing test or in the boot of the original profile.

**Before the repaste** the daily profile was 57x at 1.340 V (E-cores 45x, then 46x from 13 November). Its proof: 3 x 30 minutes of Cinebench plus GPU heat, back to back, 0 errors
(9,880 / 9,875 / 9,877; water 37.6 to 39.5 C, package 413 to 418 W, hottest core P7 at 103 / 105 / 106 C; Tj Max is 115 C here, no throttling).
One corrected error at that profile in about 28 hours: 13 November 23:05, internal parity error on P4's second thread, at light load two minutes after a fallback-training boot.

## E-cores and ring (13 November)

These ran as steps of the RAM sweep, on top of the RAM settings kept at that moment.
Each step had to pass two 7-minute y-cruncher memory runs and then 10 minutes of Cinebench with GPU heat.

| Step | Outcome | What happened |
|---|---|---|
| E-cores 46x | **PASS, kept** | Both memory runs clean. Cinebench 10 min with GPU heat: 10,059.55, 0 errors, P7 at 105 C, water 31.4 to 37.1 C. Measured E-core clock 4,580 MHz |
| E-cores 47x | **FREEZE** | Hard freeze about 3 minutes into the first memory run. No blue screen, no WHEA record, only an unclean-shutdown event. The owner held the power button |
| Ring 51x at 1.340 V | **FAIL** | Both memory runs clean (71.1 / 71.7 ns). Cinebench with GPU heat crashed at 2:24, no WHEA record, P7 at 101 C |
| Ring 51x at 1.350 V | **FAIL** | Both memory runs clean (70.3 / 70.6 ns). Cinebench with GPU heat crashed at 4:13, no WHEA record, P7 at 106 C, package 429 W |
| Ring 51x at 1.360 V (14 Nov) | **FAIL** | Both memory runs clean. Cinebench with GPU heat crashed at 3:35, no WHEA record, P7 at 105 C, package 428 W |
| Ring 52x, E-cores 48x | not tested | |

Verdicts:

- **E-cores stay at 46x.** 47x did not fail a test, it froze the machine. That is a harder kind of failure than the research guessed (see [../ram/predictions.md](../ram/predictions.md)).
- **The ring stays at 50x.** 1.340 V lasted 2:24, 1.350 V 4:13, 1.360 V 3:35. More voltage did not help within the owner's ceiling. After the repaste ring 51 passed at 1.360 V and was still not kept. See below.
  Ring 51 passed the memory test twice at both voltages. Its latency gain was about 0.3 ns at most, which is inside run-to-run noise.
  It only failed under Cinebench with the GPU heating the loop. A memory test alone would have kept it.
- Score: 10,059.55 against 9,884.75 for the like-for-like 10-minute run at E 45x on 12 November. That is +1.8%.
  The E 46x run also had the tuned RAM under it, so we cannot say how much is E-cores and how much is RAM.

## 14 November: the hard reset under a memory test

During an at-rest memory test with y-cruncher on all threads and GPU heat beside it, the PC hard-reset 5 minutes in.
The owner's photo just before: package 459 W, P6 107 C, P4 106 C, P7 105 C, water 35.8 C, GPU 536 W.

- No stop code with crash dumps on, no WHEA record, the memory checks clean twice before it. So a CPU or platform fault under heat, not memory.
- y-cruncher drew about 45 W more than the Cinebench runs that validated 57x / 1.340 V. The cores were about 5 C beyond anything validated.
- An earlier reset 25 minutes into a 60-minute y-cruncher run had the same signature. It is probably the same thing.
- After this the owner asked for sensible loading: no GPU heat in memory tests, the CPU kept cool, and a script stop at 100 C on the hottest core. Tj Max stays at 115 C in the BIOS.

## After the repaste (15 November)

Each step: the memory runs, then Cinebench with GPU heat. 10 minutes in the search, 30 minutes for the voltage steps and the confirmations.
All on the final RAM set, E-cores 46x. Measured, from the ledger.

| Step | Min | Outcome | What happened |
|---|---|---|---|
| 58x all-core, 1.340 V | 10 | **FAIL** | Crashed at 2:11 (code 0xC0000096, privileged instruction). One corrected error, on P6. Hottest core P3 at only 92 C |
| 58x all-core, 1.350 V | 10 | **FAIL** | Crashed at 4:00, no WHEA record. P3 94 C |
| 58x all-core, 1.360 V | 10 | **FAIL** | Crashed at 6:48, no WHEA record. P3 96 C. 1.360 V is the owner's ceiling |
| 58 on P0 P1 P3 P7, 57 on the rest, 1.340 V | 10 | PASS | 10,132.71, P3 96 C. Measured clocks matched the mix |
| same mix, ring 51, 1.340 V | 10 | **FAIL** | Crashed at 8:31 |
| same mix, ring 51, 1.350 V | 10 | **FAIL** | Crashed at 2:15: sooner with more voltage |
| same mix, ring 51, 1.360 V | 10 | PASS | 10,220.45, P3 97 C, 431 W |
| same mix, ring 51, 1.360 V (confirm) | 30 | PASS | 10,178.10, P3 98 C, 436 W, WHEA 0. **Not kept** (see below) |
| same mix, ring 50, 1.340 V | 30 | **FAIL** | Crashed at 7:29, one corrected error on P3. The 10-minute pass above was luck |
| 58 on P0 P1 P7, 57 on the rest, ring 50, 1.340 V | 30 | **FAIL** | Finished (10,187.46) but one corrected error on P0 |
| 57x all-core, ring 50, 1.340 V | 30 | PASS | 10,065.68, 411 W, P3 96 C, water to 39.2 C |
| 57x all-core, ring 50, 1.330 V | 30 | PASS | 10,150.45, 405 W, P3 96 C, water to 39.4 C |
| 57x all-core, ring 50, 1.320 V | 30 | PASS | 10,151.53, 401 W, P3 94 C, water to 39.9 C |
| 57x all-core, ring 50, 1.310 V | 30 | **FAIL** | Finished (10,131.17) but 2 corrected errors on P2 |
| 57x all-core, ring 50, 1.320 V (confirm) | 30 | PASS | 10,179.94, 402 W, P3 96 C, water to 40.1 C, WHEA 0. **Kept** |

What it says:

- **58x all-core is a silicon limit at or below 1.360 V.** More voltage only delayed the crash: 2:11, 4:00, 6:48. The cores were 92 to 96 C, about 10 C cooler than before the repaste, so heat is not the reason.
- **58x on some cores is out at ring 50 / 1.340 V.** P3 failed, then P0 with P3 held at 57. P0 had never logged an error before.
- **Ring 51 was not kept.** It needs 1.360 V, the ceiling. Latency on our meter: 71.3 ns against 71.1 ns at ring 50, so no measurable gain. Cinebench: about 0.9%. The owner chose ring 50 and a lower voltage.
- **Two configurations passed 10 minutes and failed 30 minutes** (the 4-core mix at 1.340 V, and before the repaste 57x at 1.320 and 1.330 V). A 10-minute pass can be luck.

### Cooler silicon lowered the voltage floor

Same test (57x all-core, ring 50, 30 minutes of Cinebench with GPU heat), before and after the repaste:

| | Voltage | Package | Hottest core | Water max | Outcome |
|---|---|---|---|---|---|
| Before (14 Nov) | 1.340 V | 421 W | P7 106 C | 36.6 C | PASS |
| Before (12 Nov) | 1.330 V | 410 W | P7 102 C | 38.6 C | FAIL, crash at 15:24 |
| Before (12 Nov) | 1.320 V | 399 W | P7 98 C | 37.8 C | FAIL, error on P2 at 26:13 |
| After (15 Nov) | 1.340 V | 411 W | P3 96 C | 39.2 C | PASS |
| After | 1.320 V | 401 W | P3 94 C | 39.9 C | PASS (twice) |
| After | 1.310 V | 396 W | P3 94 C | 40.0 C | FAIL, 2 errors on P2 |

The floor moved from 1.340 V to 1.320 V, on warmer water. The owner predicted this before the runs; the runs confirmed it.
Note that the 14 November row had E-cores at 46x and the 12 November rows 45x.

### Where the WHEA records fell (15 November)

All are corrected errors (id 19), mapped to the test windows by time:

| Time | Core | Window |
|---|---|---|
| 07:09 | P6 | 58x all-core, 1.340 V (FAIL) |
| 10:55 | P3 | 4-core 58 mix, ring 50, 1.340 V (FAIL) |
| 11:43 | P0 | 3-core 58 mix, 1.340 V (FAIL) |
| 14:26 and 14:35 | P2 | 57x, 1.310 V (FAIL) |
| 17:05 (two, 3 s apart) | P4 and P3 | boot of the original profile (59x, 1.360 V) for the before benchmark |

None falls inside a window on the final set.

## Per-core failure map

Which core logged the first error, by setting (all fixed Vcore):

| Setting | First failing core(s) | Clean cores |
|---|---|---|
| 59x all, 1.360 V, cool water | P2, then P4 | |
| 59x all, P2 and P4 held out | P6, P3, P5 | P0, P1, P7 (5 min) |
| 59x on P0/P1/P7, 58x rest, 1.360 V, GPU heat | P7 (hottest core) | all five 58x cores |
| 59x on P0/P1, 58x rest, 1.360 V, GPU heat | crash with no WHEA record | |
| 58x all, 1.360 V, GPU heat | P2 at 9:27 | |
| 58x all, 1.340 V, GPU heat | P2 at 1:56 | |
| P2 57x, rest 58x, 1.340 V | P4 at 1:46 | |
| P2 and P4 57x, rest 58x, 1.340 V | P5 at 2:20 | |
| 57x all, 1.300 V, 10 min | P2 | |
| 57x all, 1.320 V, 30 min | P2 at 26:13 | |
| 57x all, 1.340 V, 3 x 30 min | none | all |
| After the repaste: 58x all, 1.340 V | P6 at 2:11 | |
| After the repaste: 58 on P0 P1 P3 P7, ring 50, 1.340 V, 30 min | P3 at 7:29 | |
| After the repaste: 58 on P0 P1 P7, ring 50, 1.340 V, 30 min | P0 | |
| After the repaste: 57x all, 1.310 V, 30 min | P2 | |
| After the repaste: 57x all, 1.320 V, 2 x 30 min | none | all |

Weakness order at 59x, matching the 120-day error counts: **P2, P4, then P6 / P3 / P5, and P7 when hot.**
P0 and P1 never logged an error. But 59x on P0/P1 alone still crashed Cinebench with no WHEA record, so 59x is not used anywhere.

Worth knowing: P4 is one of the two factory favoured cores (fused max ratio 62), yet it is the second weakest under all-core load.
Fused ratios tell you nothing about stability under heat. Test every core.

## Voltage floors

| Ratio | Floor, 10 min hot | Floor, 30 min hot | Notes |
|---|---|---|---|
| 59x all | none at or below 1.380 V | | 1.380 V did not help P2 (3:34 vs 3:37 at 1.360 V) |
| 58x all | above 1.360 V | | P2 fails at 84 C core at 1.340 V, at 89 C at 1.360 V |
| 57x all | 1.310 V | **1.340 V** | 10-minute floor is about 30 mV optimistic |
| 57x all, after the repaste | | **1.320 V** | 1.310 V failed 30 min. 10 mV margin |
| 58x all, after the repaste | above 1.360 V | | crashes at 2:11, 4:00, 6:48 for 1.340 / 1.350 / 1.360 V |

Per-core 58x needs on P2, P4 and P5 at 1.340 V: all three fail. P4 holds 58x at 1.360 V but not at 1.340 V.

## Runs before the ledger (11 November, standalone Cinebench, no GPU heat)

| Time | Setting | Water | Outcome |
|---|---|---|---|
| 19:57 | 59x all, 1.360 V | 28.2 C (GPU at about 470 W from other work) | FAIL after about 20 s. 5 errors: P2 x4, P4 x1 (instruction TLB). Cinebench crashed |
| 20:01 | intended P2/P4 excluded, but Cinebench reset its own affinity | 28.5 C | FAIL after about 15 s: P2 x1, P4 x2. Counts as a repeat of the baseline |
| 20:03 | P2 and P4 held out for real | 28.7 to 31.9 C | FAIL at 5:18: P6 at 3:45, P3 at 3:58, P5 at 4:39. P0, P1, P7 clean |

## Every ledger run

From Shakedown's ledger. Minutes is the planned length (0 = one pass). Water is start to max. Power is package average.
Errors are corrected CPU errors (WHEA).

| Run | Setting | Min | Result | Reason | Score | Errors | Failed | Hottest | Water C | Pkg W | GPU W |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 12 Nov 03:48 | 59x all, 1.360 V, 18 C water | 0 | PASS | | 9,822.64 | 0 | | P3 80 C | 17.9 to 20.8 | 415 | 70 |
| 03:54 | 59x all, 1.360 V, from 18.7 C water | 10 | FAIL | crashed at 5:06 | | 3 | P2 P4 | P7 83 C | 18.5 to 22.2 | 409 | 73 |
| 04:18 | 58x all, 1.325 V (A-Tuning, not verified) | 10 | stopped | stopped by hand at 7:38 | | 0 | | P7 77 C | 17.5 to 21.7 | 379 | 70 |
| 04:27 | 59x, ring 45x, 1.360 V | 10 | FAIL | crashed at 1:31, no WHEA | | 0 | | P7 82 C | 21.2 to 21.4 | 385 | 201 |
| 04:29 | 59x, ring 45x, 1.380 V | 10 | FAIL | crashed at 4:21 | | 1 | P2 | P7 91 C | 21.6 to 25.5 | 420 | 276 |
| 04:34 | 59x, ring 45x, 1.380 V, P2 held out | 10 | FAIL | crashed at 0:23 | | 0 | | P3 82 C | 25.7 | 388 | 522 |
| 04:40 | P0 P1 P7 59x, rest 58x, 1.360 V, GPU heat | 10 | FAIL | crashed at 4:31 | | 4 | P7 | P7 92 C | 19.2 to 29.8 | 416 | 528 |
| 04:53 | P0 P1 59x, rest 58x, 1.360 V, GPU heat | 10 | FAIL | crashed at 3:26, no WHEA | | 0 | | P7 92 C | 22.9 to 31.3 | 416 | 531 |
| 05:00 | 58x all, 1.360 V, GPU heat part of the run | 10 | PASS | | 9,921.00 | 0 | | P3 93 C | 29.4 to 33.0 | 414 | 194 |
| 05:12 | 58x all, 1.340 V, GPU heat | 10 | FAIL | crashed at 4:23 | | 6 | P2 | P7 93 C | 27.5 to 33.5 | 419 | 536 |
| 05:36 | 58x all, 1.360 V, GPU heat | 10 | FAIL | finished, 2 errors at 9:27 | 9,973.62 | 2 | P2 | P7 97 C | 23.6 to 34.7 | 425 | 536 |
| 05:50 | 57x all, 1.360 V, GPU heat | 10 | PASS | | 9,800.58 | 0 | | P7 98 C | 28.3 to 35.3 | 409 | 538 |
| 06:08 | 57x all, 1.340 V, GPU heat | 10 | PASS | | 9,884.75 | 0 | | P7 96 C | 25.9 to 35.3 | 403 | 537 |
| 06:30 | 57x all, 1.330 V, GPU heat (sweep) | 10 | PASS | | 9,870.05 | 0 | | P7 96 C | 25.9 to 36.1 | 397 | 538 |
| 06:42 | 57x all, 1.320 V, GPU heat, no cooldown | 10 | PASS | | 9,873.59 | 0 | | P2 93 C | 35.1 to 36.5 | 392 | 539 |
| 06:54 | 57x all, 1.310 V, GPU heat, no cooldown | 10 | PASS | | 9,879.22 | 0 | | P7 93 C | 35.6 to 36.6 | 386 | 539 |
| 07:06 | 57x all, 1.300 V, GPU heat, no cooldown | 10 | FAIL | crashed at 8:03 | | 1 | P2 | P7 92 C | 35.4 to 36.1 | 376 | 538 |
| 07:23 | 57x all, 1.320 V, 30 min, run 1 of 3 | 30 | (no verdict) | clean for 31:48, verdict lost to a tooling fault | about 9,879 avg | 0 | | | to 37.4 | | |
| 07:57 | 57x all, 1.320 V, 30 min, run 2 of 3 | 30 | FAIL | finished, 1 error at 26:13 | 9,824.51 | 1 | P2 | P7 98 C | 32.4 to 37.8 | 399 | 532 |
| 08:31 | 57x all, 1.330 V, 30 min, run 1 of 3 | 30 | PASS | | 9,851.66 | 0 | | P7 101 C | 33.7 to 38.4 | 405 | 533 |
| 09:03 | 57x all, 1.330 V, 30 min, run 2 of 3 | 30 | FAIL | crashed at 15:24, no WHEA | | 0 | | P7 102 C | 37.8 to 38.6 | 410 | 534 |
| 09:19 | 57x all, 1.340 V, 30 min, run 1 of 3 | 30 | PASS | | 9,879.54 | 0 | | P7 103 C | 37.6 to 39.2 | 413 | 535 |
| 09:52 | 57x all, 1.340 V, 30 min, run 2 of 3 | 30 | PASS | | 9,874.50 | 0 | | P7 105 C | 38.6 to 39.4 | 415 | 535 |
| 10:24 | 57x all, 1.340 V, 30 min, run 3 of 3 | 30 | PASS | | 9,877.21 | 0 | | P7 106 C | 38.9 to 39.5 | 418 | 535 |
| 18:31 | P2 57x, rest 58x, 1.340 V (BIOS), GPU heat | 10 | FAIL | crashed at 2:32 | | 4 | P4 | P7 100 C | 27.6 to 32.9 | 411 | 530 |
| 18:41 | P2 P4 57x, rest 58x, 1.340 V (BIOS), GPU heat | 10 | FAIL | load exited with an access violation at 5:58 | | 3 | P5 | P7 101 C | 27.3 to 35.0 | 415 | 537 |
| 13 Nov 10:31 | 57x, E 45x, **ring 51x**, 1.340 V, GPU heat | 10 | FAIL | crashed at 2:24, no WHEA | | 0 | | P7 101 C | 31.1 to 34.4 | 414 | 534 |
| 10:52 | 57x, **E 46x**, ring 50x, 1.340 V, GPU heat | 10 | PASS | | 10,059.55 | 0 | | P7 105 C | 31.4 to 37.1 | 417 | 534 |
| 15:17 | 57x, E 46x, **ring 51x, 1.350 V**, GPU heat | 10 | FAIL | crashed at 4:13, no WHEA | | 0 | | P7 106 C | 32.0 to 36.4 | 429 | 533 |
| 14 Nov, about 00:37 | 57x, E 46x, **ring 51x, 1.360 V**, GPU heat | 10 | FAIL | crashed at 3:35, no WHEA | | 0 | | P7 105 C | 30.8 to 35.1 | 428 | |
| 14 Nov, about 06:46 | 57x, E 46x, ring 50x, 1.340 V, GPU heat (validation) | 30 | PASS | | 10,028.45 | 0 | | P7 106 C | 31.5 to 36.6 | 421 | |
| 15 Nov 07:07 (after the repaste) | **58x all**, E 46x, 1.340 V, GPU heat | 10 | FAIL | ended with code 0xC0000096 at 2:11 | | 1 | P6 | P3 92 C | 31.8 to 35.4 | 418 | 535 |
| 07:24 | 58x all, 1.350 V, GPU heat | 10 | FAIL | ended with an access violation at 4:00 | | 0 | | P3 94 C | 30.9 to 35.8 | 428 | 539 |
| 07:42 | 58x all, 1.360 V, GPU heat | 10 | FAIL | ended with an access violation at 6:48 | | 0 | | P3 96 C | 31.1 to 36.8 | 430 | 537 |
| 08:05 | 58 on P0 P1 P3 P7, 57 rest, 1.340 V, GPU heat | 10 | PASS | | 10,132.71 | 0 | | P3 96 C | 31.2 to 37.3 | 415 | 538 |
| 08:29 | same mix, ring 51x, 1.340 V, GPU heat | 10 | FAIL | crashed at 8:31 | | 0 | | P3 94 C | 31.5 to 37.2 | 417 | 538 |
| 08:52 | same mix, ring 51x, 1.350 V, GPU heat | 10 | FAIL | crashed at 2:15 | | 0 | | P3 92 C | 31.7 to 35.3 | 425 | 535 |
| 09:09 | same mix, ring 51x, 1.360 V, GPU heat | 10 | PASS | | 10,220.45 | 0 | | P3 97 C | 31.9 to 37.9 | 431 | 534 |
| 09:26 | same mix, ring 51x, 1.360 V, GPU heat (confirm) | 30 | PASS | | 10,178.10 | 0 | | P3 98 C | 30.4 to 39.1 | 436 | 540 |
| 10:48 | same mix, ring 50x, 1.340 V, GPU heat | 30 | FAIL | crashed at 7:29 | | 1 | P3 | P3 96 C | 32.2 to 38.1 | 420 | 533 |
| 11:11 | 58 on P0 P1 P7, 57 rest, ring 50x, 1.340 V, GPU heat | 30 | FAIL | finished, 1 error | 10,187.46 | 1 | P0 | P3 96 C | 32.5 to 39.0 | 421 | 535 |
| 11:59 | **57x all**, ring 50x, 1.340 V, GPU heat | 30 | PASS | | 10,065.68 | 0 | | P3 96 C | 32.8 to 39.2 | 411 | 537 |
| 12:44 | 57x all, 1.330 V, GPU heat | 30 | PASS | | 10,150.45 | 0 | | P3 96 C | 33.1 to 39.4 | 405 | 539 |
| 13:29 | 57x all, 1.320 V, GPU heat | 30 | PASS | | 10,151.53 | 0 | | P3 94 C | 33.5 to 39.9 | 401 | 537 |
| 14:19 | 57x all, 1.310 V, GPU heat | 30 | FAIL | finished, 2 errors | 10,131.17 | 2 | P2 | P3 94 C | 33.6 to 40.0 | 396 | 540 |
| 14:59 | 57x all, **1.320 V**, GPU heat (confirm, final set) | 30 | PASS | | 10,179.94 | 0 | | P3 96 C | 31.8 to 40.1 | 402 | 541 |

Notes on individual runs:

- 04:27: the GPU heat in use was an early pixel-shader build that drew only about 117 W and stole CPU time. Treat the GPU column for 04:27 to 04:34 with care.
- 04:34: holding P2 out did not save 59x at ring 45x. Ring 45x did not help at all.
- 05:00: GPU heat was switched off part way, so this pass is milder than the 1.360 V fail at 05:36.
- 13 to 15 November rows: the RAM under these runs was the tuned set of that hour, not XMP. See [../ram/results.md](../ram/results.md).
  E-cores 47x has no ledger row: it froze in the memory test before Cinebench started.
- 05:12: package power was about the same as at 1.360 V, so at the time we could not confirm A-Tuning had applied 1.340 V. Later screen captures confirmed each A-Tuning step.

## Scores

Scores move by under 1% between passing runs at the same setting. Higher voltage at the same ratio did not raise the score.
58x scored 9,921 to 9,974. 57x scored 9,800 to 9,885 with E 45x and XMP RAM, and 10,059.55 with E 46x and the tuned RAM.
After the repaste, 57x with E 46x and the final RAM scored 10,066 to 10,180 in 30-minute runs. Ring 51 added about 0.9%.

## Before and after (15 November, Measured)

| | Original profile (59x, 1.360 V, E 45x, ring 50x, XMP) | Final set |
|---|---|---|
| Latency, our meter (best of 3) | 74.1 ns | 71.2 ns |
| Read bandwidth | 126.4 GB/s | 128.1 GB/s |
| y-cruncher bench 2.5b | 117.75 s | 114.72 s |
| Cinebench 2026 multi | did not finish: stalled at 30 minutes, then again at 6 minutes, no result | 10,215.28 |
| Cinebench 2026 single | 553.7 | 540.04 |

Water about 28 C at the start of each. One run of each Cinebench test.
The original profile logged two corrected errors (on P4 and P3) within seconds of each other right after it booted.
Single-core is 2.5% slower at 57x. Everything that touches memory is faster. And the profile we started from was not stable.
A run's score depends on how much of it ran with GPU heat and how hot the water got, so compare scores only between like runs.
