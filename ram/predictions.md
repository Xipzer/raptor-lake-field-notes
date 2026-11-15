# Predictions vs reality

On the morning of 13 November, after the no-POST, we stopped and researched every step still in the queue.
For each one we wrote down what we expected, the rule behind it, and whether the gain was worth the risk.
Then the owner asked us to run them anyway, one clock at a time. This page compares the two.

Status: 15 November 2026, morning. The RAM tune is finished, and the long runs have had their say.

## Sources used for the predictions

| Key | Source |
|---|---|
| PAK-DDR5 | M. Pakhtunov, DDR5 Memory tuning and optimization on Intel LGA1700/LGA1851, rev. 5.04: https://pakt.cc/en/ddr5/ , timings: https://pakt.cc/en/ddr5/timings/ , formulas: https://pakt.cc/en/ddr5/formulas/ , PDF: https://pakt.cc/files/DDR5_5.04_2026_EN.pdf |
| PAK-CPU | M. Pakhtunov, Adaptive tuning and optimization of Intel processors, v2.01: https://pakt.cc/files/CPU_2.01_2026_EN_FINAL.pdf |
| HYNIX | SK hynix DDR5 32 Gb M-die datasheet Rev 1.2: https://uttc.com.tw/wp-content/uploads/2026/01/DDR5_SDRAM_32Gb_M-die_Rev.1.2-1_H5CG58MHBDX051N.pdf (the 32 Gb die; we found no public 24 Gb M-die datasheet) |
| OCN-INTEL | overclock.net Intel DDR5 thread, post 1 (quoting Arni90): https://www.overclock.net/threads/official-intel-ddr5-oc-and-24-7-daily-memory-stability-thread.1794772/ |
| OCN-LIGHT | overclock.net Nova / Lightning owners' thread: https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/ |
| IFX | integralfx DDR4 OC guide: https://github.com/integralfx/MemTestHelper/blob/oc-guide/DDR4%20OC%20Guide.md |
| BZ | Buildzoid on tRFC divided by tREFI: https://www.patreon.com/buildzoid/posts/low-effort-rank-77403831 |
| IGOR | igor'sLAB, JEDEC vs Intel DDR5 specs: https://www.igorslab.de/en/intel-vs-jedec-ddr5-timings-in-extreme-technical-practice-test/ |
| SKAT | SkatterBencher #67, the shared core/ring voltage rail: https://skatterbencher.com/2023/12/16/skatterbencher-67-intel-core-i9-14900k-overclocked-to-6200-mhz/ |
| INTEL | Intel root cause statement: https://community.intel.com/t/intel-core-13th-and-14th-gen-desktop-instability-root-cause-update/127016 |
| C&C | Chips and Cheese on Alder Lake E-cores and ring clock: https://chipsandcheese.com/p/alder-lake-e-cores-ring-clock-and-hybrid-teething-troubles |

The overclock.net threads could only be read as archived copies of their first pages. Claims from them are thin.

## The table

Predicted outcome kinds: PASS, TEST-FAIL (test error or crash under load), FALLBACK (training falls back, our NO TRAIN),
BOOT-CRASH (trains, then Windows crashes at boot), POST-HANG (needs hands).

| Step | Predicted | Rule behind the prediction | Advice at the time | What happened |
|---|---|---|---|---|
| tRFCpb 560 | PASS likely | Pakhtunov's formula gives about 554 | Yes, cheap | PASS |
| tRFCpb 460 | Coin flip | Arni90's ratio (0.69 of tRFC) is about 470 | Only if 560 shows a gain | PASS |
| tRFCpb 420 | **TEST-FAIL or FALLBACK** | Below every published ratio | Skip | **PASS** |
| tRAS 84 | PASS | Above tRCD + tRTP | Yes | PASS |
| tRAS 76 | PASS likely | Same | Yes | PASS |
| tCWL 38 at tCL 40 | PASS likely | DRAM's own CWL is CL minus 2 | Yes | PASS |
| tREFI 40960 | PASS likely at 55 C or less | Retention falls with heat | Yes | PASS at 52.5 / 57.8 C |
| tREFI 49152 | Coin flip at 55 to 57 C | Same | Yes, if 40960 is clean | PASS at 52 / 57.8 C in short runs. 49151 later FAILED TestMem5 at 61.5 C |
| tREFI 65535 | **TEST-FAIL, late, when warm** | Same. It had failed before | Skip until DIMMs run about 45 C | PASS in every short run at up to 59 C. **FAIL in long TestMem5 runs at about 60 C.** The prediction was right |
| tCL 38 at DIMM 1.45 V | TEST-FAIL | Already seen in the controls | No | FAIL |
| tCL 38 at DIMM 1.50/1.48 V | PASS cold, coin flip at 55 C+ | CL scales with VDD on M-die | Marginal. Pick this or a higher tREFI, not both | PASS at 53 / 58.8 C, with tREFI 65535 in place |
| tRFC 670 | Coin flip | About 160 ns M-die floor. 630 fell back | Skip | PASS |
| tRFC 660 | **TEST-FAIL or FALLBACK** | Same | Skip | **PASS** |
| tRCD/tRP 49 | **About 50% BOOT-CRASH** | tRCD plateau. 48 crashed at boot | Skip | **PASS** |
| tRRD_L 11 | **BOOT-CRASH likely** | 10 gave an NMI at boot | Skip | **PASS** |
| tWR 22 | Coin flip | "tRTP follows tWR down to about 10" | Skip | PASS. The rule was wrong |
| tWR 20 | **TEST-FAIL likely** | Same | Skip | **PASS**. tRTP still reads 23 |
| E-cores 46 | PASS likely | 46 is a common setting on this CPU | Optional | PASS |
| E-cores 47 | Coin flip, PASS or TEST-FAIL | 47 is a "good chip" target | Skip | **FREEZE**. A kind of failure the prediction did not list |
| Ring 51 | Coin flip | Stock ceiling is 50. HT on lowers it | Try | FAIL at 1.340, 1.350 and 1.360 V |
| Ring 52 | TEST-FAIL likely | Same | Skip | not tested |

And one that was never predicted, because it happened before the research:

| Step | Expected | What happened |
|---|---|---|
| tRDRD_sg 14 | A soft failure at worst. 15 had just passed twice | **NO POST.** No picture, five manual power cycles |

## The score

- 20 predictions were tested.
- 7 were confident and right (6 passes and the tCL 38 fail at 1.45 V).
- 8 were coin flips, counting tRCD/tRP 49 at "about 50% boot crash". Six of those passed, ring 51 failed, and E-cores 47 froze the PC.
- **5 were confident predictions of failure, and all 5 passed the short runs**: tRFCpb 420, tREFI 65535, tRFC 660, tRRD_L 11, tWR 20.
  Four of them are in the final set. **tREFI 65535 failed TestMem5 at about 60 C** in runs of 16 to 50 minutes, so that prediction was right after all.
- Every short-run miss was on the pessimistic side. The one step predicted to pass that later failed was tREFI 49152, and only in a long, hot test.
- The advice was to skip nine of the tested steps. Eight of those passed. The ninth was E-cores 47, which froze the PC: that advice was right.
- The two worst events of the day were not on the list at all: the no-POST came from a step nobody had flagged, and the freeze was a failure kind the prediction did not allow for.

"Passed" in the table means two 7-minute runs unless it says otherwise. The long validation moved tREFI 65535 and 49152 back to the other column.

## Why the predictions were too dark

1. **They leaned on our own bad data.** The old failures behind several predictions were measured on a marginal base (tCL 38 at 1.45 V),
   on hot DIMMs without a fan, and in jumps of two clocks. tREFI 65535 and tRFC 670 "had failed before". On a stable base they pass.
2. **A single step on a stable base is a different experiment from a two-step jump on hot sticks.**
   tRCD/tRP 48 crashed at boot, straight from 50, before the DIMM fan and on the 1.45 V rails.
   tRCD/tRP 49, one clock from 50, on the tuned base with the fan, passed. The same goes for tRRD_L 10 (from 12) and 11.
   Then 48 and 10 were retried one clock at a time on the good base. Both crash-looped at startup again. The predictions for 49 and 11 were too dark; the original crashes were real.
3. **Published temperature limits are other people's kits, but they are not nothing.** The 45 to 50 C ceiling for tREFI 65535 on 24 Gb M-die is widely repeated. Ours passed short runs at 57.5 C and failed long ones at about 60 C. The limit was real; our short runs just could not see it.
4. **One prediction rested on a "fact" we had got wrong ourselves.** We had logged that this BIOS derives tRTP from tWR.
   The research took that from our journal and concluded tWR 22 would push tRTP below the DRAM minimum.
   A read-back showed tRTP at 23, its XMP value. Our error went into the research and came back as a prediction.
5. **The same wrong tRTP gave a wrong tRAS floor.** The research computed tRCD + tRTP as 50 + 11 = 61. The real floor was 50 + 23 = 73.
   The tRAS 84 and 76 predictions were right anyway. A step to 72 would have been planned on a wrong floor.

## What the research got right

- The order: voltages first, tRAS and tRFCpb as cheap early steps, tCWL 38 at tCL 40 as a free step, tREFI in stages.
- tCL 38 needs DIMM voltage, not CPU-side voltage. It failed at 1.45 V and passed at 1.50/1.48 V.
- "Any E-core instability will look like a memory error." E-cores 47 died inside the memory stress test, not in Cinebench.
- The ring. A 50x ceiling with Hyper-Threading on is what the most detailed source runs on the same CPU. Ours failed at 51x twice.
- The failsafe cannot recover a POST hang or an early crash. We had just learned that the hard way, and learned it again with the freeze.
- It said to confirm the tRTP read-back before trusting the tWR rule. We did that hours later than we should have.

## Lessons

1. **Predictions are a guide, not a verdict.** Use them to order the work and to decide who needs to be in the room. Do not use them to skip the test.
2. **Predict the kind of failure, not only pass or fail.** What matters for an unattended run is whether a failure needs hands. Neither of the two steps that needed hands was flagged for it.
3. **Go one clock at a time near a known failure,** from a base that has passed a control, with the DIMMs cooled.
4. **When a prediction rests on a rule, test the rule first.** One read-back of tRTP would have removed two wrong predictions and one wrong floor.
5. **Pessimistic misses are cheap. Optimistic misses are not.** Five wrong "it will fail" calls cost us nothing but a delay. One unflagged step cost a no-POST.
6. **Judge a prediction by the long test, not the short one.** On the short runs we scored tREFI 65535 as a miss. The long, hot runs proved it right. Grade predictions only once the validation is done.
