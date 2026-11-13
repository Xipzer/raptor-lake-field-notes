# CPU results

All runs on Projector. Load: Cinebench 2026.1.3 multi-core. E-cores 45x and ring 50x unless noted.
"GPU heat" means the RTX 5090 held at about 530 W in the same loop. Water is read at the pump (the coolest point).

## Result

**Daily profile: 57x all P-cores, E 45x, ring 50x, fixed 1.340 V, LLC Level 1.** Now in the BIOS, confirmed by a SCEWIN export.

- Proof: 3 x 30 minutes of Cinebench plus GPU heat, back to back, 0 errors. Scores 9,880 / 9,875 / 9,877.
  Water 37.6 to 39.5 C, package 413 to 418 W, hottest core P7 at 103 / 105 / 106 C (Tj Max is 115 C here, no throttling).
- Cost against 59x: 3.4% clock. Measured all-core cost of 58x to 57x: about 0.7% in Cinebench. Games: estimate 0 to 3%, depending on how CPU-bound they are.

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

Notes on individual runs:

- 04:27: the GPU heat in use was an early pixel-shader build that drew only about 117 W and stole CPU time. Treat the GPU column for 04:27 to 04:34 with care.
- 04:34: holding P2 out did not save 59x at ring 45x. Ring 45x did not help at all.
- 05:00: GPU heat was switched off part way, so this pass is milder than the 1.360 V fail at 05:36.
- 05:12: package power was about the same as at 1.360 V, so at the time we could not confirm A-Tuning had applied 1.340 V. Later screen captures confirmed each A-Tuning step.

## Scores

Scores move by under 1% between passing runs at the same setting. Higher voltage at the same ratio did not raise the score.
58x scored 9,921 to 9,974. 57x scored 9,800 to 9,885.
A run's score depends on how much of it ran with GPU heat and how hot the water got, so compare scores only between like runs.
