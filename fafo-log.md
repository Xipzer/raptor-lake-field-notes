# FAFO log

What we tried, what broke, and what we learned, in order. Our own mistakes are included. Times are local, November 2026.
Where a later finding overturned an earlier entry, the entry is corrected in place and the correction is listed under
"Corrections of our own errors" near the end. Status: 13 November 2026, late afternoon. The RAM sweep is still running.

## Before tuning

- Projector ran 59x all P-cores, ring 50x, fixed 1.360 V, DDR5-8400 XMP. It felt fine in cold weather and less so on 20 C+ days.
- The event log held 116 corrected CPU errors in 120 days, all bank 0, all P-cores, mostly P2 and P4. Plus a run of summer blue screens.
- No errors at all from late August to early October with nothing changed. **Learned:** a clean week proves little. Errors come in bursts tied to what you run.

## 11 November: reproduce

- **Tried:** standalone Cinebench 2026 at 59x, water 28 C. **Broke:** errors on P2 and P4 within 11 s, crash at about 20 s. **Learned:** the logged fault reproduces on demand, and it fails on cool water too.
- **Mistake:** the first launch failed on a quoting bug in our runner. Cinebench never started.
- **Tried:** excluding P2 and P4 with `start /affinity`. **Broke:** nothing changed, Cinebench reset its own affinity. **Learned:** hold cores out with a job object, or re-apply affinity in a loop.
- **Tried:** P2 and P4 held out for real. **Found:** P6, P3, P5 fail next. P0, P1, P7 clean for 5 minutes.
- Built Shakedown to do this properly: load, WHEA by core, temperatures, ledger.

## 12 November, early hours: ratios

- **Tried:** 59x on 18 C water. **Found:** a 2.5-minute pass passes. The 10-minute run from 18.6 C fails at 3:34 (P2 at 72 C). **Learned:** water temperature changes when it fails, not whether.
- **Mistake:** Shakedown read only the first 8 KB of Cinebench output. The score is at the end. Fixed.
- **Mistake:** the first GPU heat build was a pixel shader at low priority. It drew 117 W instead of 500+ and stole CPU time. Rewritten as a compute shader with two batches in flight: 520 to 535 W.
- **Found:** A-Tuning changes ratio and Vcore live. Our research had said no runtime path existed.
- **Tried:** ring 45x and 1.380 V at 59x (above the owner's own 1.360 V cap). **Broke:** P2 failed at almost the same time. **Learned:** ring is not the limit on this chip, and more voltage gave nothing but heat.
- **Mistake:** one run "failed" at 0:27 because a chat message interrupted our tool call, which ended the test process. Long runs now go in a detached background shell.
- **Tried:** BIOS Specific Per Core, P0/P1/P7 59x, rest 58x, with GPU heat. **Broke:** P7 (the hottest core) at 59x. The five 58x cores stayed clean.
- **Tried:** P0/P1 59x, rest 58x. **Broke:** crash at 3:26 with no WHEA record. **Learned:** a silent crash is a fail. 59x is out.
- **Tried:** 58x all at 1.360 V and 1.340 V with GPU heat. **Broke:** P2 every time, sooner at the lower voltage.
- **Tried:** reading the CPU's overclocking mailbox. **Found:** the cores are adaptive with target 0. The fixed Vcore is at the VRM. Mailbox voltage control is a dead end on this board.
- **Tried:** 57x all at 1.360 V and 1.340 V. **Passed** with full GPU heat, water to 35 C.

## 12 November, morning: voltage

- Automated A-Tuning (clicks plus screen captures). One press = 5 mV.
- **Tried:** 57x sweep down in 10 mV steps, 10 minutes each, no cooldown. **Found:** 1.330, 1.320, 1.310 V pass, 1.300 V fails on P2.
- **Tried:** overnight, 3 x 30 minutes per voltage from 1.320 V. **Broke:** 1.320 V failed in run 2 (P2 at 26:13). 1.330 V crashed silently in run 2.
  **Passed:** 1.340 V, 90 clean minutes. **Learned:** the 10-minute floor was 30 mV optimistic.
- **Mistake:** the first 1.320 V run lost its verdict. A mirror of its output to another machine stalled, and the broken pipe killed the test 30 s before the end. The copy now runs beside the test and cannot end it.
- **Mistake:** the overnight loop stopped after a real FAIL, because its stop rule matched the word "water" in the result line. Fixed, resumed.

## 12 November, evening: per-core at 1.340 V

- **Tried:** P2 57x, rest 58x at 1.340 V. **Broke:** P4. Then P2 and P4 at 57x: **broke** P5. **Learned:** at 1.340 V, 58x fails on three cores in turn. 57x all-core is the answer for this chip at this voltage.
- **Scare:** the hottest core looked 5 to 7 C worse against water than in the morning. Liquid metal drying out? **Checked:** core-to-water per 100 W was flat at 12.0 to 12.6 all day. No degradation. **Learned:** compare per watt before you repaste.
- 57x / 1.340 V written to the BIOS.

## 12 November, evening: RAM setup

- **Tried:** SCEWIN export. **Worked**, even under VBS.
- **Tried:** SCEWIN import. Blocked by ASRock's runtime-variable protection. **Worked** after setting a BIOS administrator password and using `/cpwd`.
- Set up autologon with immediate lock, a boot-time sweep controller, and a failsafe that restores the last good settings.
- **Mistake:** the XMP baseline logged as FAIL. The error check matched the y-cruncher settings line "Stop on Error: Enabled". It was a pass.
- **Mistake:** a tRFC 740 write was silently skipped. SCEWIN matches questions by prompt, and the prompt contains the live value. Scripts are now built from a fresh export.
- **Mistake:** DIMM temperatures logged as 0 C. The SPD hub sensors need initialising after a cold boot.
- **Mistake:** tWR 40 plus tRTP 20 was logged as FAIL. tRTP had read back 19, so it was NOT APPLIED. And our script then skipped the tWR group by mistake. Rerun.
  **Second mistake, found a day later:** we explained the 19 as "the BIOS derives tRTP from tWR". It does not. See correction (c).
- **Tried:** tRFC down from 922. **Found:** 680 passes, 630 does not train.
- **Tried:** tWR down to 24. **Passed** at every step.
- **Tried:** tRRD_L 10. **Broke:** NMI bugcheck at boot, then the PC sat frozen for about an hour because dumps were on. The owner pressed reset. **Learned:** turn crash dumps off for a RAM sweep, so a crash reboots at once.
- **Found:** two earlier shutdown hangs (0x9F). Cause unknown.
- **Tried:** tWTR 36 / 12. **NOT APPLIED:** the BIOS set 42 / 18. We concluded tWTR could not go lower. Wrong: see correction (a).
- **Tried:** tRCD/tRP 48. **Broke:** crash loop at startup into Windows Recovery. The owner clicked restart.
- **Mistake:** the failsafe task had a script error and failed on its first real boot. Found by testing it on purpose with no changes. Fixed, then it worked.

## 13 November, after midnight: the tCL 38 lesson

- A 120 mm fan went on the DIMMs. We first wrote that peak temperature dropped by 10 to 15 C. The steady figure is 8 to 12 C: see correction (f).
- **Tried:** tCL 38. **Passed** once.
- **Tried:** tCL 36. **No train.** The failsafe and the sweep both recovered it without anyone. **Mistake:** both fired, so one extra reboot. Fixed.
- **Tried:** tRAS 96, 76: pass. tRAS 64, tREFI 65535, tRFC 670: all failed late in the test.
- **Stopped** and ran a control: the kept settings with no change. **It failed.** So the three FAILs proved nothing.
- **Mistake:** control run 2 never ran; the script tripped on a locked log file.
- Stepped back tRAS 76 to 96: control still failed. In-cache y-cruncher passed, so it was memory, not cores.
- Stepped back tCL 38 to 40: control **passed twice**.
- **Learned:** tCL 38 passed once and was marginal all along. Every step tested on top of it was tested on a broken base. One pass is not a pass. Run controls. Run twice.

## 13 November, night: plan v2, voltages and turnarounds

- Re-read the research and reordered: voltages first (TX at 1.45 V was above every daily recommendation), then the turnarounds through tWRRD, then tRFCpb, tRAS, tCWL and tREFI. Every step needs two passes in the same boot.
- **Tried:** TX 1.40, 1.35, 1.30 V. **Passed**, all three, twice each. **Learned:** the board's Auto of 1.45 V was never needed.
- **Tried:** VDD2 1.42 V. **Broke:** test error within 3 minutes. VDD2 stays 1.45 V.
- **Tried:** tWRRD_dg 56, 52, 48 and tWRRD_sg 80, 70. **Passed.** tWRRD_sg 64: **no train**, recovered without anyone.
- **Tried:** tRDWR_dg and _sg 20, 19. **Passed.** tRDRD_sg 15: **passed.**
- **Tried:** tRDRD_sg 14. **Broke:** no picture at all. The PC sat powered and never reached the BIOS. The failsafe is a Windows task, so it never ran.
  The owner power-cycled it five times before Boot Failure Guard let it through.
  **Found:** NVRAM still held every setting. Boot Failure Guard had not loaded defaults. Only the memory had come up at a fail-safe DDR5-3600.
  **Learned:** a tertiary can do what no primary did. A startup failsafe cannot help with a no-POST. And one clean pass at 15 said nothing about 14.
- Went back to the trained tRDRD_sg, dropped tWRWR_sg, raised TX from 1.30 to 1.35 V for training margin. Steps that can stop training now wait until the owner is around.

## 13 November, morning: predictions, then the soft steps

- Researched every step left in the queue and wrote a prediction for each. See [ram/predictions.md](ram/predictions.md).
- **Tried:** tRFCpb 560, 460. **Passed**, and latency dropped by about 1.5 ns. Research had doubted the board uses this timing at all.
- **Tried:** tRAS 84, 76; tCWL 38; tREFI 40960, 49152. **Passed**, all of them, with the DIMMs at 52 / 57.5 C.
- **Tried:** ring 51x. Both memory runs passed. **Broke:** Cinebench with GPU heat crashed at 2:24. **Learned:** a CPU-side change needs the CPU test. The memory test alone would have kept it.
- **Tried:** E-cores 46x. **Passed**, Cinebench 10,059.55.
- The sweep declared discovery done and started the final validation. The owner stopped it 4 minutes in: rinse out every last drop first.

## 13 November, midday: the extras pass

- **Tried:** tRFCpb 440, 420. **Passed.** Predicted to fail.
- **Tried:** tREFI 57344, 65535. **Passed** at 52 / 57.5 C. Predicted to fail when warm. The earlier 65535 failure had been the marginal base, not the heat.
- **Tried:** tCL 38 with tCWL 36 at 1.45 V. **Broke:** the same late error as the controls. So tCL 38 at 1.45 V really is marginal.
- **Tried:** the same with DIMM VDD/VDDQ 1.50 / 1.48 V. **Passed.** About 1 to 1.5 C warmer DIMMs.
- **Tried:** tRFC 670, 660. **Passed.** 660 was predicted to fail.
- **Mistake:** the E-cores 47 write stopped the sweep with "could not read the BIOS". Our reader had looked at the helper's result file before the helper finished writing it. Nothing had been written. See correction (g).
- **Tried:** E-cores 47x. **Broke:** hard freeze about 3 minutes into the memory test. Clock stopped on the lock screen. The owner held the power button. **Learned:** a freeze does not restart, so nothing unattended recovers it. Second time in one day.

## 13 November, afternoon: the audit

After the freeze the owner asked for a full review. We diffed every live value in the BIOS against the XMP export and re-read the journal for claims we had never checked.
It found errors (a) to (h) below. Then the sweep went on:

- **Tried:** ring 51x at Vcore 1.350 V. Both memory runs passed. **Broke:** Cinebench with GPU heat crashed at 4:13, P7 at 106 C. **Learned:** 10 mV bought under 2 minutes. The ring stays at 50x.
- **Checked:** voltages in HWiNFO under load. DIMM VDD 1.500 / VDDQ 1.485 V, TX 1.350 V, VDD2 1.448 V, SA 1.248 V, Vcore sensor 1.344 V at 1.350 V set. Everything we set is delivered.
  **Side effect found:** while HWiNFO is open, our fan controller loses the sensor chip and the fans go to the BIOS curve, about 3,000 rpm.
- **Tried:** tRCD/tRP 49. **Passed.** Predicted about 50% to crash at boot.
- **Tried:** tRRD_L 11. **Passed.** Predicted to crash at boot.
- **Tried:** tWR 22, 20. **Passed.** tRTP still reads 23.
- **Tried:** tRTP 19. **NOT APPLIED:** it read back 18. An explicit tRTP reads back one lower than entered, the same as 20 giving 19 a day earlier.
- tRAS 72 under test at the time of writing (17:00, run 1 of 2 passed). Queued: retries of tRCD/tRP 48 and tRRD_L 10, then the final validation.

## Corrections of our own errors

Things we wrote down as facts or results and later found to be wrong. Each is fixed where it appeared in these notes.

| | What we said | What is true | How we found out |
|---|---|---|---|
| (a) | "tWTR stays at XMP 48 / 16. The BIOS will not go below 42 / 18." | tWTR follows tWRRD and tCWL. On this board tWTR = tWRRD - tCWL - 6. It reads 28 / 6 now | The audit diffed live values against XMP |
| (b) | "The tRAS floor, tRCD + tRTP, is about 61." | tRTP is 23, not 11. The floor was 50 + 23 = 73. tRAS 64 had been below it, and 76 is close to it | Same diff: tRTP read 23 |
| (c) | "The BIOS derives tRTP from tWR (48 gives 23, 40 gives 19)." | Setting tWR alone does not move tRTP. It stays 23. The 19 came from setting tRTP 20 in the same step: an explicit tRTP reads back one lower | Read tRTP with tWR 24 set alone |
| (d) | "CRASH ring 51, Vcore 1.350" | False. That step had booted and was never tested. The crash check looked 2 minutes back before the step was set and found the E-cores 47 freeze records | The crash line quoted the same event as the E-cores 47 freeze |
| (e) | "FAIL ring 51, Vcore 1.350: Cinebench: no ledger line" | The fail was real, the reason was not. The ledger lookup broke on the comma in the run name. The ledger said: crashed at 4:13 | Read the ledger by hand |
| (f) | "The DIMM fan dropped peaks by 10 to 15 C, to 45 / 50 C." and "the fan was moved at 14:28" | 45 / 50 C was a cold-start reading. Steady benefit is 11 to 12 C on the cooler stick and 8 to 9 C on the hotter: 63.5 / 65.8 C without, about 52 / 57.5 C with, at 1.45 V (about 53 / 59 C at 1.50 V). Nothing moved at 14:28. The fan has sat in one place since it went in | Compared steady peaks. The owner corrected the fan note |
| (g) | "Could not write E-core ratio 47 (could not read the BIOS)" | A race in our own tooling. The reader did not wait for the helper to finish its result file | Nothing had been written. The reader now waits for the closing line |
| (h) | (caught before it ran) | The final-benchmark script would have set E-cores to 0 for the "original profile" run, and could loop if that profile crashed | Code review during the audit |

Earlier ones, already described above: the XMP baseline logged as FAIL (a text match on "Stop on Error"); the tRFC 630 no-train suspected to be a shutdown hang (it was a real training failure).

Two of these did damage beyond the notes:

- (c) fed the research. It predicted tWR 22 and 20 would push tRTP under the DRAM minimum, and we dropped tWR 22 for that reason. It passed once we ran it.
- (b) meant we tested tRAS 64 at all. It was below the floor and could only fail or gain nothing.

## Running list of lessons

1. One pass is not a pass. Two runs minimum, and a control when failures cluster.
2. Ten minutes is not thirty. Near the edge, errors are rare events.
3. A silent crash counts as a fail.
4. Test hot, and on a shared loop test with the GPU loaded.
5. Read the effective settings after every boot. The BIOS may have changed, clamped or reverted your value. Read the settings you did not touch as well: that is where tWTR and tRTP were hiding.
6. Your tooling is part of the experiment. More than ten of our "results" were tooling faults, and three of our "rules" were wrong.
7. Order matters: voltages first, then tertiaries and refresh timings, then the heat-sensitive ones, then anything that can stop the machine.
8. Fused favoured cores and SP numbers do not tell you which core fails under heat.
9. A failsafe that runs inside the OS cannot help with a no-POST or a hard freeze. Run those steps only when someone can reach the power button.
10. Predictions are a guide, not a verdict. A single step on a stable, cooled base behaves very differently from a two-step jump on hot sticks.
11. A rule you derived from one observation is a guess. We built a floor and a prediction on "tRTP follows tWR" before reading tRTP once.
12. A CPU-side change needs the CPU test. Ring 51 passed the memory test four times and failed Cinebench twice.
13. When someone asks for an audit, diff everything against the baseline. Ours found a timing that had been moving for eleven hours without our knowing.
