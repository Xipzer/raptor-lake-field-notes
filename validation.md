# Validation: how to know a setting is stable

How we tested, what fooled us, and the rules we ended with. It applies to CPU and DDR5 tuning on any recent Intel desktop, and most of it to AMD as well.
Labels: **Measured** (our logs), **Source** (public, linked), **Estimate** (our reasoning).

## Quick reference

| Pitfall | What it did to us | Rule |
|---|---|---|
| [10 minutes is not 30](#run-length-10-minutes-overstates-the-floor) | The 10-minute voltage floor was 30 mV too optimistic | Search with short runs, confirm the floor with 30-minute runs |
| [One pass is luck](#run-counts-and-what-zero-failures-proves) | Mixes that passed 10 minutes failed 30 | 1 run to screen, 3 to trust, 10 or more for the final |
| [A marginal base poisons every step](#control-runs) | Three FAILs blamed on innocent steps | Run the unchanged base when failures cluster |
| [Cold tests pass hot faults](#heat-is-part-of-the-test) | tREFI passed 60 short runs, failed at 60 C | Test at the temperature real use reaches |
| [y-cruncher is a launcher](#y-cruncher-pitfalls) | Affinity masks ignored | Mask the worker process, after 20 s |
| [TestMem5 coverage](#testmem5-pitfalls) | A run tested 40% of memory | Free memory first, read the coverage line |
| [Parsers lie](#testmem5-pitfalls) | A clean 60-minute run logged as FAIL | Know the exact pass and fail wording |
| [CPU heat inside a RAM test](#keep-the-cpu-out-of-a-ram-test) | 459 W, a core at 107 C, hard reset | Keep the CPU cool while testing RAM |
| [WHEA windows too short](#whea-accounting) | "0 errors" said when 37 had been logged in 30 days | Count over the whole period, by core |
| [DRAM errors never reach WHEA](#whea-accounting) | Every RAM failure had WHEA 0 | A clean log proves nothing about the DIMMs |
| [Crash dumps on during a sweep](#crash-dumps-on-or-off) | A blue screen froze the PC for an hour | Dumps off for unattended sweeps, small dumps on otherwise |
| [Crash dumps off hide causes](#crash-dumps-on-or-off) | Two days of "slow training" that was not | Turn small dumps back on as soon as you can |
| [Slow boots misread](#slow-boots-0x9f-shutdown-hangs-not-memory-training) | 0x9F shutdown hangs read as memory training | Check for a 0x9F record first |

## Run length: 10 minutes overstates the floor

**Measured, 12 November (57x, Cinebench 2026 with GPU heat on the shared loop):**

- A 10-minute sweep down in 10 mV steps passed 1.330, 1.320 and 1.310 V and failed 1.300 V.
- Then 3 x 30 minutes per voltage: 1.320 V failed in run 2 (P2 at 26:13), 1.330 V crashed silently in run 2, 1.340 V passed 90 clean minutes.
- So the 10-minute floor was **30 mV optimistic**.

**Measured, 15 November (after a liquid-metal repaste):** a per-core 58x mix passed 10 minutes at 1.340 V and failed at 7:29 in a 30-minute run. Another mix finished 30 minutes with one corrected error.
57x all-core then passed 30 minutes at 1.340, 1.330 and 1.320 V, and failed at 1.310 V (two corrected errors on P2). 1.320 V passed a second 30-minute run. Final: 1.320 V, 10 mV above a failure in the hardest test we run.

Why (Estimate): near the edge, errors are rare events. The chance of seeing one grows with time under load and with heat. The water keeps warming for 20 minutes or more on a shared loop.

Public: CoreCycler and most overclocking guides recommend hours per core for final validation ([CoreCycler](https://github.com/sp00n/corecycler)). We found **no public measurement** of the 10 against 30 minute gap in millivolts; ours is one chip.

## Run counts and what zero failures proves

What we settled on: **1 run to screen, 3 runs for a reliability check, 10 to 12 for the final pass only.** Long loops at every intermediate step waste hours.

Zero failures in n runs does not mean zero risk. If each run fails with probability p, the 95% upper bound on p after n clean runs is 1 - 0.05^(1/n) (roughly 3/n, the "rule of three", [Wikipedia](https://en.wikipedia.org/wiki/Rule_of_three_(statistics))):

| Clean runs | Failure rate still possible (95%) |
|---|---|
| 1 | 95% |
| 3 | 63% |
| 10 | 26% |
| 16 | 17% |
| 60 | 5% |

So 8 runs cannot tell "fails 5 times in 8" from "fails 3 times in 8", and a single pass near the edge tells you very little. That is why we used long runs for the final word, not many short ones.

## Control runs

**Measured, 13 November.** Three RAM steps in a row failed late in a 7-minute test (tRAS 64, tREFI 65535, tRFC 670). We stopped and ran the kept settings with nothing changed:

| Control | Settings | Result |
|---|---|---|
| A | the kept set (tCL 38, tRAS 76) | FAIL |
| B | A with tRAS back to 96 | FAIL; an in-cache y-cruncher run with no memory traffic passed, so it was memory, not cores |
| C | B with tCL back to 40 | PASS, twice |

tCL 38 at DIMM 1.45 V had passed once and was marginal. Every step after it had been tested on a broken base. Two of the three "failed" steps later passed on a stable base.

Rules:

- When failures cluster, run a control before believing any of them.
- Write down how you will read the control before it runs, so you cannot fit the story afterwards.
- A failed step proves the step failed only if the base still passes.

## Heat is part of the test

- **CPU:** on a shared loop, heat the water with the GPU while the CPU test runs. We used a D3D11 compute load at about 520 to 535 W on the GPU ([overclock-autopilot](https://github.com/Xipzer/overclock-autopilot), "GPU heat"). 59x passed a cold 2.5-minute run on 18 C water and failed every 10-minute run.
- **RAM, the main lesson of the campaign (Measured, 14 November):** tREFI 65535 passed about 60 short y-cruncher runs and two 60-minute y-cruncher runs. It then failed TestMem5 three times with the hotter stick at 59 to 61.5 C. tREFI 49151 failed too. 32767 passed four 60-minute TestMem5 runs at the same heat.
  Looser tRFC (688) and tRFCpb (460) did not fix it. Only the refresh interval did. The two sticks ran 6 to 9 C apart, and the hotter one set the limit.
- **Source:** tREFI is the most temperature-sensitive DDR5 timing; reports put the limit for 24 Gb M-die around 45 to 50 C at tREFI 65535 ([overclock.net temperature thread](https://www.overclock.net/threads/ddr5-most-temperature-sensitive-timings.1808660/); [r/overclocking scaling test](https://www.reddit.com/r/overclocking/comments/1odgz3l/investigating_ddr5_trefi_scaling_with_high/)). The public guidance was closer to the truth than our short runs.
- Log DIMM temperature on every memory test (both sticks, peak and at the first error). The SPD hub sensors read 0 after a cold boot until a tool such as HWiNFO initialises them.

## y-cruncher pitfalls

- **The launcher is not the worker.** `y-cruncher.exe` starts a child binary from `Binaries\` (for example `14-BDW ~ Kurumi.exe`) that does the work. Set affinity on the child, not the launcher.
  Source: [Rynez Stability Monitor README](https://github.com/exidum79/Rynez-Stability-Monitor) ("that file is just a launcher"); [r/overclocking](https://www.reddit.com/r/overclocking/comments/18frn4v/how_to_run_ycruncher_standalone_as_efficiently_as/) names the child binary.
- **Wait about 20 s before setting the mask.** A mask set in the first instant was undone by y-cruncher's own start-up. Read per-core load back to check. Ending the launcher can leave the worker running, so end the worker.
- **Do not match "error" in its output.** The settings banner prints "Stop on Error: Enabled". Our first parser logged the XMP baseline as FAIL for that line.
- **It names the thread that found the bad result, not the part that caused it.** A checksum error "on an E-core" needed memory traffic: an in-cache run (`-M:2G`, small tests) passed on the same settings. Use an in-cache run to tell a core fault from a memory fault.
- What we ran: `stress -M:30G -D:60 -TL:420 FFTv4 N63 VT3` (about 7 minutes, 30 GB) per sweep step, and 60 minutes at checkpoints. It found every access-timing fault within minutes and missed the heat-bound refresh fault entirely.

## TestMem5 pitfalls

- **Coverage drops if memory is not free.** Started straight after another memory test, TM5 tested 40% of memory. Check the "Testing N MB x 32 (P%)" line; expect about 75 to 82%. On a WSL host drop the Linux page cache first (see [windows.md](windows.md#wsl)).
  Source: the CoolCmd build notes that the old 0.12 build could report success after allocating far less memory than asked ([CoolCmd/TestMem5](https://github.com/CoolCmd/TestMem5)).
- **Confine it once, at launch, never mid-run.** It runs as 32 worker processes. Moving them mid-run disturbs the test.
- **0.13.1 writes only "Testing completed" on a clean pass.** Our parser expected other wording and logged a clean 60-minute run as FAIL.
- **"An error N occurred in the program" is a failure.** On unstable settings it is most likely memory corruption. We first treated it as "not a result".
- With the anta777 config it found the tREFI fault in 16 to 50 minutes, and it heats the sticks about 3 C more than y-cruncher.

## The at-rest test (memhold)

Fill 30 GiB with patterns, lock it in physical memory, leave it, and re-check it every 2 minutes for 45 minutes, with a small y-cruncher run beside it.
A mismatch is re-read after a cache-line flush, so a wrong value really stored in the DIMM is told apart from a bad read. Proven with an injected bit before use.
A flip in data nobody touched points straight at refresh. It never saw the tREFI fault, because it ran with the sticks near 50 C, not 60 C. Run it hot.
Source code: [raptor-lake-overclock-autopilot](https://github.com/Xipzer/raptor-lake-overclock-autopilot) `tools/memhold.c`. We found **no public tool** that does exactly this on Windows.

## Keep the CPU out of a RAM test

**Measured, 14 November:** the at-rest test with y-cruncher on all threads and GPU heat beside it drew 459 W and put one core at 107 C. The PC hard-reset 5 minutes in, with no stop code although dumps were on. That is a CPU or platform fault under heat, not a memory result.
Since then: no GPU heat during RAM tests, the load sized up from a few threads while watching the hottest core, and a guard that ends the load at 100 C on the hottest core.

## Test CPU-side changes with the CPU test

Ring 51 passed the memory test four times and failed Cinebench with GPU heat twice. E-cores 47x froze the PC 3 minutes into a memory test.
Any ring, E-core or Vcore change needs the CPU load as well as the memory load.

## WHEA accounting

- **Count over the whole period, not the test window.** On 12 November we quoted "0 WHEA" from a few short windows when 37 corrected errors had been logged in the last 30 days. Before tuning, the log held 116 in 120 days (P2 47, P4 37, P3 15, P6 12, the rest spread).
- **Map each record to a core.** WHEA-Logger event 19 carries an APIC ID. On a 14900K(S): P-core index = APIC / 8 (P0..P7 = APIC 0, 8 .. 56); E-cores are APIC 64, 66 .. 94. That map is a free weakness ranking before you change anything.
- **What the IDs meant here:** 19 = corrected CPU error (ours were all MCA bank 0: instruction TLB and internal parity). 1 = "fatal hardware error" on the boot after a failed DRAM training; every one of our eight sat on such a boot, none came from a test. 3 = informational after a timing change.
- **Map each record to a test window.** On 15 November all seven corrected errors of the day sat inside failing tests or the original profile's boot; none on the final set. A count relayed from the other machine said nine; two were from earlier days.
- **Silent crashes count as fails.** Several crashes left no WHEA record at all (59x on P0/P1, ring 51 at every voltage, 58x all-core at 1.350 and 1.360 V).
- **DDR5 on-die ECC is invisible.** It corrects single-bit errors inside the chip and reports nothing to the OS ([ATP](https://www.atpinc.com/tw/blog/ddr5-what-is-on-die-ecc-how-is-it-different-to-traditional-ecc)). Every RAM failure in our sweep had WHEA 0.
- Source on reading event 19: [overclock.net WHEA 19 thread](https://www.overclock.net/threads/about-whea-event-19-warnings.1536052/).

## Crash dumps on or off

- **Dumps on during a sweep, Measured 12 November:** tRRD_L 10 crashed at boot with bugcheck 0x80 (NMI). Windows sat frozen on the blue screen for about an hour, writing nothing, until a manual reset.
  Writing a dump means running the OS on the memory that just failed. Public reports of blue screens stuck at "0%" with no dump written are common ([Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/3799557/blue-screen-of-death-not-generating-dump-file); [Super User](https://superuser.com/questions/1637479/what-could-cause-bsod-to-stuck-at-0-resulting-no-memory-dump)). We found no public source that ties the freeze to unstable RAM specifically; that link is ours (Estimate, from one event).
- **Dumps off hid the cause of slow boots for two days** (see the next section).
- **What we do now:** dumps off and automatic restart on only while an unattended sweep runs; small dumps (CrashDumpEnabled 3) at all other times. Keep a list of everything you change for a sweep (dumps, `bcdedit recoveryenabled`, `bootstatuspolicy`, startup tasks, paused services) and put it all back at the end.
- With dumps on, a reset with **no** stop code and no WHEA record points at the CPU or platform (a hard reset), not a memory bugcheck.

## Slow boots: 0x9F shutdown hangs, not memory training

- During the sweep many restarts took 323 to 353 s instead of about 45. We read them as "slow memory training when hot", built a rule on it, and dropped or flagged values for it. **That was wrong.**
- With dumps back on, a slow restart left bugcheck 0x9F DRIVER_POWER_STATE_FAILURE, parameter 1 = 3: a device held a power IRP too long during shutdown, and Windows waited for its watchdog. A fixed timeout, hence the same length every time.
- All three dumps named the NVIDIA driver on the RTX 5090. Every hang lined up with a large local model server (about 31 GB of GPU memory) being torn down mid-shutdown. Closing GPU-heavy programs before a scripted restart stopped them.
- **Tell them apart:** a failed training shows the fail-safe values (tRFC 395, DDR5-3600 and so on, see [scewin.md](scewin.md#8-after-a-training-fallback-the-prompts-change)). A shutdown hang shows your own values, a 0x9F record (WER-SystemErrorReporting 1001) and a long gap between shutdown and the next start. Check the 1001 timestamp; do not blame an older one.
- **Source:** [Microsoft bug check 0x9F reference](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x9f--driver-power-state-failure); NVIDIA driver 0x9F reports with the same subcode on other cards ([NVIDIA developer forum](https://forums.developer.nvidia.com/t/bug-nvlddmkm-sys-0x9f-driver-power-state-failure-subcode-3-when-entering-s4-hibernate-d3-set-power-irp-never-completes-rtx-4060-windows-10-22/386712); [Microsoft Q&A](https://learn.microsoft.com/en-us/answers/questions/5935800/constant-bsod-driver-power-state-failure-0x9f-fail)). We found **no public report** of this being mistaken for DDR5 training time.

## Event log cheat sheet

| Event | Meaning for tuning |
|---|---|
| WHEA-Logger 19 | Corrected CPU error. APIC ID names the logical CPU |
| WHEA-Logger 1 | Here: the boot after a failed DRAM training (fallback), not a test crash |
| Kernel-Power 41 | Unclean restart. BugcheckCode 0 with dumps off says nothing; with dumps on it means a hard reset |
| EventLog 6008 | Unexpected shutdown. Its time is the last heartbeat, not the crash time |
| WER-SystemErrorReporting 1001 | A bugcheck, with code and parameters |
| Kernel-General 12 / 13, User32 1074 | Boot / shutdown / who asked for the restart |
| Display 4101, nvlddmkm 13 / 14 / 153 | GPU driver reset. Matters if a GPU load runs beside the CPU test |
| Application 1000 / 1002 | The load itself crashed or hung |

A normal boot gap here is about 45 s. More than about 120 to 150 s: check for a 0x9F first, then for fallback values.

## After any unclean stop

`chkdsk /scan C:` (online, read only), `sfc /scannow`, and the Linux file systems if you run WSL (`dmesg | grep EXT4-fs`). Then confirm by export that the BIOS runs the last good set before the next step.
All of ours were clean except the first `sfc`, which repaired 62 Start Menu shortcut files and nothing else. After the tCL 36 corruption we also checked the signatures of the boot files: valid on disk, so the corruption had happened in RAM while loading.

## The final ladder we used

RAM (15 November, DIMM fan on, no GPU heat): at-rest 45 minutes with 30 GiB held, y-cruncher VT3 60 minutes, TestMem5 anta777 60 minutes. Then CPU: Cinebench 2026 30 minutes with GPU heat, twice, at the final voltage. Then hot reboots.
Results are in [ram/results.md](ram/results.md) and [cpu/results.md](cpu/results.md).
