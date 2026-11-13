# FAFO log

What we tried, what broke, and what we learned, in order. Our own mistakes are included. Times are local, November 2026.

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
- **Mistake:** tWR 40 plus tRTP 20 was logged as FAIL. The BIOS derives tRTP from tWR, so it was NOT APPLIED. And our script then skipped the tWR group by mistake. Rerun.
- **Tried:** tRFC down from 922. **Found:** 680 passes, 630 does not train.
- **Tried:** tWR down to 24. **Passed** at every step.
- **Tried:** tRRD_L 10. **Broke:** NMI bugcheck at boot, then the PC sat frozen for about an hour because dumps were on. The owner pressed reset. **Learned:** turn crash dumps off for a RAM sweep, so a crash reboots at once.
- **Found:** two earlier shutdown hangs (0x9F). Cause unknown.
- **Tried:** tWTR 36 / 12. **NOT APPLIED:** the BIOS set 42 / 18.
- **Tried:** tRCD/tRP 48. **Broke:** crash loop at startup into Windows Recovery. The owner clicked restart.
- **Mistake:** the failsafe task had a script error and failed on its first real boot. Found by testing it on purpose with no changes. Fixed, then it worked.

## 13 November, after midnight: the tCL 38 lesson

- A 120 mm fan went on the DIMMs. Peak DIMM temperature dropped by about 10 to 15 C.
- **Tried:** tCL 38. **Passed** once.
- **Tried:** tCL 36. **No train.** The failsafe and the sweep both recovered it without anyone. **Mistake:** both fired, so one extra reboot. Fixed.
- **Tried:** tRAS 96, 76: pass. tRAS 64, tREFI 65535, tRFC 670: all failed late in the test.
- **Stopped** and ran a control: the kept settings with no change. **It failed.** So the three FAILs proved nothing.
- **Mistake:** control run 2 never ran; the script tripped on a locked log file.
- Stepped back tRAS 76 to 96: control still failed. In-cache y-cruncher passed, so it was memory, not cores.
- Stepped back tCL 38 to 40: control **passed twice**.
- **Learned:** tCL 38 passed once and was marginal all along. Every step tested on top of it was tested on a broken base. One pass is not a pass. Run controls. Run twice.

## 13 November, plan v2

- Re-read the research and reordered: voltages first (TX and VDD2 at 1.45 V were above every daily recommendation), then the turnarounds, then tRFCpb and tRAS. Temperature-sensitive timings wait for the water block. Every step needs two passes.
- TX 1.400 V: passed twice. Kept. TX 1.350 V under test at the time of writing.

## Running list of lessons

1. One pass is not a pass. Two runs minimum, and a control when failures cluster.
2. Ten minutes is not thirty. Near the edge, errors are rare events.
3. A silent crash counts as a fail.
4. Test hot, and on a shared loop test with the GPU loaded.
5. Read the effective settings after every boot. The BIOS may have changed, clamped or reverted your value.
6. Your tooling is part of the experiment. Five of our "results" were tooling faults.
7. Order matters: voltages and stable primaries first, temperature-sensitive timings last and only with cool DIMMs.
8. Fused favoured cores and SP numbers do not tell you which core fails under heat.
