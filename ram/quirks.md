# RAM quirks

## The 14th-gen "SA bug"

- Some 14th-gen chips hard-freeze (no blue screen) once VCCSA goes above a chip-specific point, typically about 1.12 to 1.30 V.
  VT3 and TM5 can then crash instantly. It caps memory frequency, because higher speeds need more SA.
- Reports: "1.28v SA max before freezing" on a 14900KS (OCN-INTEL page 1307); "anything above 1.232V freezes during TM5" (page 1319);
  some say microcode 0x129 or later eased it (pages 1482, 1637). See the [SA freeze thread](https://www.overclock.net/threads/14900k-14900k-cpu-sa-voltage-issue-freezing-problem-and-solved.1808469/).
- Not documented by Intel.
- How to check: at a known stable profile, raise SA in 10 to 20 mV steps with a short VT3 pass each. A freeze that appears at one step and goes away a step lower is the signature.
- **Status here: not tested yet.** Our SA is a manual 1.250 V. We have no reason to raise it at 8400.

## tWTR is set through tWRRD, and it moves when you are not looking

- On Intel, tWTR_S and tWTR_L follow tWRRD_dg and tWRRD_sg (Arni90, OCN-INTEL post 1; integralfx).
- **Measured on this board: tWTR = tWRRD - tCWL - 6.** tWTR_S uses tWRRD_dg and tWTR_L uses tWRRD_sg.

| State | tWRRD_dg / sg | tCWL | tWTR_S / tWTR_L shown |
|---|---|---|---|
| XMP | 62 / 94 | 40 | 16 / 48 |
| Kept set | 48 / 70 | 36 | 6 / 28 |

- So tWTR is **not** at its XMP value in our kept set. It dropped when we lowered tWRRD, and it rose again a little when we lowered tCWL.
  For a while our notes said "tWTR stays 48 / 16, the BIOS will not go lower". That was wrong. See [../fafo-log.md](../fafo-log.md).
- Our direct write of tWTR 36 / 12 came back as 42 / 18 after boot, with no retrain. Both came back 6 higher than entered.
  ASRock owners report the same 6-cycle offset between typed and shown values (OCN-INTEL post 32688; OCN-LIGHT post 3789).
- The public formula has a different constant (tWTR = tWRRD - tCWL - 10 in [Pakhtunov's guide](https://pakt.cc/en/ddr5/formulas/)). Use the one your own board shows.
- Tune tWRRD_sg/dg, leave tWTR on Auto, and read tWTR back after any change to tWRRD or tCWL.

## tWR does not move tRTP. tRTP is its own setting

- We first wrote that this BIOS derives tRTP from tWR. **It does not.**
- The evidence we misread: one step set tWR 40 and tRTP 20 together, and tRTP read back 19. We took the 19 as "derived from tWR 40".
- The read-backs that settle it: with tWR 24 set alone, tRTP read 23. With tWR at 20 it still reads 23. 23 is the XMP value.
- The 19 came from setting tRTP 20. An explicit tRTP reads back one clock lower than entered: 20 gave 19, and later 19 gave 18.
  Our sweep counts a read-back that differs from the entry as NOT APPLIED and does not test it. So tRTP is still untuned at 23.
- Consequence: there is no tWR floor for tRTP's sake. We skipped tWR 22 for hours for a reason that did not exist.

## The tRAS floor is tRCD + tRTP, with the tRTP you read back

- Rule (integralfx; Pakhtunov): tRAS below tRCD + tRTP does not shorten the read path. It gains nothing and can fail.
- We computed the floor as 50 + 11 = about 61, with a tRTP we had estimated from the wrong tWR rule above.
- The read-back tRTP is 23. The floor was 50 + 23 = **73**, and is 49 + 23 = 72 now that tRCD is 49.
- So tRAS 76 is close to the floor, and the tRAS 64 we tested on 13 November was below it.
- Never compute this from an assumed tRTP. Read it back.

## ASRock BIOS behaviours we hit

- **A pending value can be reverted by the BIOS.** After the tRRD_L 10 crash, the BIOS showed tRRD_L back on Auto before anyone touched it.
- **Boot Failure Guard does not always load defaults.** The manual says it restores defaults after a number of failed boots (ours: 4; configurable 2 to 250).
  After our no-POST and five manual power cycles it showed its message and went on, but NVRAM still held every setting, including the one that caused the no-POST.
  Only the memory came up differently: DDR5-3600, tCL/tRCD 29, tRFC 395, TX 1.100 V, VDD2 1.120 V. After any rough boot, export and look. Do not assume either way.
- **Some kept settings have no live value in the BIOS.** DIMM VDD and VDDQ and the E-core ratio show only what is stored.
  We verified them elsewhere: PMIC read-outs in HWiNFO, and the measured E-core clock.
- **VDD_IMC is VDDQ_TX and VDD_CPU is VDD2.** Other owners said so; we confirmed it under load in HWiNFO (1.350 V set and read on TX, 1.450 V set and 1.448 V read on VDD2).
- **Runtime variables are password-protected.** SCEWIN import works only with a BIOS administrator password and `/cpwd`. See [method.md](method.md).
- **SCEWIN matches the full prompt, including the live value.** Stale scripts are silently skipped.
- **A harmless warning on every import**: "Error in writing variable TcgStorageDynamicSetupVar to NVRAM". The settings we wrote still applied.
- **VDD_CPU and VDD_IMC follow XMP voltage on Auto.** On our kit that put TX and VDD2 at 1.45 V.
- **"Realtime Memory Timing"** exists and can be enabled from SCEWIN. Whether live timing changes work on this board under VBS: untested. Live changes cannot retrain, so primaries need a reboot anyway.
- **"ASRock Timing Optimization"** is on by default. It may rewrite timings left on Auto when you change one. Set every timing you care about explicitly.

## The tRFC 395 fallback signature

When training fails, this board falls back to safe timings and boots anyway. You will see:

- tRFC and tRFC2 at **395**,
- tWR 54, tCL / tRCD / tRP 29,
- a slow boot: 135 to 282 s instead of about 45 s.

We saw it at tRFC 630, tCL 36 and tWRRD_sg 64. The last one also showed tRAS 57, TX 1.100 V and tWRRD 42 / 54.
A plain "the BIOS changed my value" (tWTR, tRTP) changes only the value you set and boots at normal speed.
That is how our script tells NO TRAIN from NOT APPLIED: did values we did not touch change?

The same fail-safe values showed up after the no-POST, at DDR5-3600. The difference is how you get there.
A NO TRAIN boots by itself after a few minutes. A NO POST sits there until someone cycles the power.

## A no-POST from one tertiary: tRDRD_sg 14

- tRDRD_sg 15 passed twice. tRDRD_sg 14 gave no picture at all. Trained is 16.
- tRDRD_sg tracks the DRAM's tCCD_L, about 5 ns, which is 21 clocks at 8400. The trained 16 is already far below that ([Pakhtunov](https://pakt.cc/en/ddr5/timings/)).
- The _sg timings give little performance next to the _dg ones (Arni90, OCN-INTEL post 1), and they are reported as heat-sensitive on M-die.
- So we went back to the trained value and dropped tWRWR_sg from the plan. Poor trade: almost no gain, and the only step all week that needed five power cycles.
- First suspects for no-POST and training loops in the same guide are VDD2, TX and SA. We had just lowered TX to 1.30 V, so we put it back to 1.35 V. We did not prove that was the cause.

## tRFCpb is worth a look

- The board's Auto was 796 clocks, which is 189.5 ns: the JEDEC value for this die size. Research was not sure this BIOS uses the timing at all.
- 560, 460, 440 and 420 (100 ns) all passed. Latency dropped by about 1.5 ns at the first step, more than any other single step.
- One owner of three 24 Gb M-die kits runs 100 ns for tRFCpb and 155 ns for tRFC2 (OCN-INTEL post 32731). Our 420 and 660 are close to that.

## DIMM temperature limits for 24 Gb M-die

| Source | Limit |
|---|---|
| [OCN temperature thread](https://www.overclock.net/threads/ddr5-most-temperature-sensitive-timings.1808660/) | tREFI 65k when under 45 to 50 C; 32767 good to 70 C |
| [r/overclocking](https://www.reddit.com/r/overclocking/comments/1odgz3l/investigating_ddr5_trefi_scaling_with_high/) | errors from about 47 C at tREFI 65535, 8600 |
| OCN-LIGHT page 6 | keep 2x24 kits below 49 C at full load |
| OCN-LIGHT page 54 | "don't like going over 42c if i want tight timings" |

Ours: 63.5 / 65.8 C steady peak without airflow, about 52 / 57.5 C with a 120 mm fan at DIMM 1.45 V, about 53 / 59 C at 1.50 V.
The fan is worth 11 to 12 C on the cooler stick and 8 to 9 C on the hotter one.

What we measured against those limits:

- tREFI 65535 failed once at 54 C. That was on a marginal base (tCL 38 at 1.45 V). On a stable base it passed twice at 52 / 57.5 C.
- tCL 38 at DIMM 1.50 / 1.48 V passed twice at 53 / 58.8 C.
- So this kit runs settings above the published comfort zone in two 7-minute runs. That is not yet a daily-use proof.
  The kit has a history of instability around 55 to 60 C in summer, at XMP.

A water block is planned; the expected result is DIMMs a few C above loop water, about 40 to 45 C under heat soak.

## DDR5 on-die ECC is invisible to WHEA

- DDR5 chips correct single-bit errors internally and say nothing ([ATP](https://www.atpinc.com/tw/blog/ddr5-what-is-on-die-ecc-how-is-it-different-to-traditional-ecc)).
- On a non-ECC desktop, DRAM-side errors never become WHEA records. They show up only as test failures, crashes or corrupt files.
- Every RAM failure in our sweep had **WHEA 0**. A clean WHEA log proves nothing about the DIMMs.
- WHEA records on this platform point at the CPU side: cores, ring or the memory controller.

## 0x9F shutdown hangs

- Two reboots during the sweep hung for 6 to 7 minutes and ended in bugcheck 0x9F DRIVER_POWER_STATE_FAILURE (a driver holding a power request).
- Cause not identified. They were at shutdown, not in a test.
- We first suspected one of them made the tRFC 630 boot look like a training failure. The same tRFC 395 fallback at tCL 36 later confirmed tRFC 630 was a real training failure.
- Fix for the sweep: crash dumps off and automatic restart on, so a hang ends in a reboot instead of a long dump write on unstable RAM.

## DIMM sensors read 0 after a cold boot

The SPD hub thermal sensors read 0 until a tool initialises them. HWiNFO does. Our own reader did not, so step 1 has no temperatures.
If your logger shows 0 C, it is not a cold DIMM.

## Errors reported on a core are not always the core

The control B failure was an FFTv4 checksum mismatch reported on an E-core. An in-cache y-cruncher run with no memory traffic passed for 8 minutes on the same settings.
The error needed memory traffic, so it was RAM or IMC, not that core. y-cruncher names the thread that found the bad result, not the part that caused it.
