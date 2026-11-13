# RAM quirks

## The 14th-gen "SA bug"

- Some 14th-gen chips hard-freeze (no blue screen) once VCCSA goes above a chip-specific point, typically about 1.12 to 1.30 V.
  VT3 and TM5 can then crash instantly. It caps memory frequency, because higher speeds need more SA.
- Reports: "1.28v SA max before freezing" on a 14900KS (OCN-INTEL page 1307); "anything above 1.232V freezes during TM5" (page 1319);
  some say microcode 0x129 or later eased it (pages 1482, 1637). See the [SA freeze thread](https://www.overclock.net/threads/14900k-14900k-cpu-sa-voltage-issue-freezing-problem-and-solved.1808469/).
- Not documented by Intel.
- How to check: at a known stable profile, raise SA in 10 to 20 mV steps with a short VT3 pass each. A freeze that appears at one step and goes away a step lower is the signature.
- **Status here: not tested yet.** Our SA is a manual 1.250 V. We have no reason to raise it at 8400.

## tWTR is set through tWRRD

- On Intel, tWTR_S and tWTR_L follow tWRRD_dg and tWRRD_sg (Arni90, OCN-INTEL post 1; integralfx).
- Our direct write of tWTR 36 / 12 came back as 42 / 18 after boot, with no retrain. The BIOS enforces a floor tied to other timings.
- ASRock owners also report the tWTR shown in the BIOS and in Timing Configurator being about 6 cycles off (OCN-INTEL post 32688; OCN-LIGHT post 3789).
- Tune tWRRD_sg/dg, and read values back with more than one tool.

## ASRock BIOS behaviours we hit

- **tRTP follows tWR.** Setting tWR 40 and tRTP 20 booted with tRTP 19. XMP: tWR 48 gives tRTP 23. Set tWR and let the BIOS derive tRTP.
- **A pending value can be reverted by the BIOS.** After the tRRD_L 10 crash, the BIOS showed tRRD_L back on Auto before anyone touched it.
- **Boot Failure Guard** loads defaults after 4 failed boots (count is configurable 2 to 250). After any rough night, confirm the BIOS is not on defaults before testing.
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

We saw it at tRFC 630 and tCL 36. A plain "BIOS clamped my value" (tWTR, tRTP) changes only the value you set and boots at normal speed.
That is how our script tells NO TRAIN from NOT APPLIED: did values we did not touch change?

## DIMM temperature limits for 24 Gb M-die

| Source | Limit |
|---|---|
| [OCN temperature thread](https://www.overclock.net/threads/ddr5-most-temperature-sensitive-timings.1808660/) | tREFI 65k when under 45 to 50 C; 32767 good to 70 C |
| [r/overclocking](https://www.reddit.com/r/overclocking/comments/1odgz3l/investigating_ddr5_trefi_scaling_with_high/) | errors from about 47 C at tREFI 65535, 8600 |
| OCN-LIGHT page 6 | keep 2x24 kits below 49 C at full load |
| OCN-LIGHT page 54 | "don't like going over 42c if i want tight timings" |

Ours: 60 to 66 C peak without airflow, 45 to 56 C with a 120 mm fan. The kit has a history of instability around 55 to 60 C in summer.
Before the fan, every passing step was above 60 C. With the fan, tREFI 65535 still failed at 54 C.
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
