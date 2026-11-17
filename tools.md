# Tools

Everything we used, and how you can reproduce the method on your own machine.
Our scripts are tied to our machines and are not published here. They are described so you can build your own.

## Shakedown (our own, to be published as `shakedown`)

A small native Windows tool for CPU stability runs. Planned as a public repository called `shakedown`.

- Runs a load you choose (Cinebench 2026 by default) once, for N minutes, or in **Rank** mode.
- Watches the WHEA log during the run and maps each corrected error to a core by APIC ID.
- Logs per-core real clock (APERF/MPERF), temperature, throttling and package power through PawnIO's signed IntelMSR module.
  An on-demand elevated helper task reads the MSRs, so the tool itself runs without admin rights.
- Closes the load when it crashes, so no error dialog waits for a click.
- **Rank**: runs the load, holds out the cores that erred (a job object affinity limit the load cannot undo), and runs again until it passes. The order is the weakness ranking.
- **GPU heat**: a Direct3D 11 compute shader that holds the GPU at full power (520 to 535 W on an RTX 5090) to heat a shared loop. Switches itself off at a water limit.
- Writes every run to a CSV ledger: note, minutes, result, reason, score, errors, failing cores, hottest core, water start and max, package power, GPU power.
- Tray icon and a small panel; a command-line build for scripts (`shakedown-cli run --minutes 10 --exclude 2,4`, `rank`, `sensors`, `whea`, `watch`).
- Reads water temperature from the pump's sensor; this part is specific to our hardware.

## CPU sweep scripts

- **A-Tuning helper**: an elevated scheduled task that reads commands from a file (`minus N`, `plus N`, `apply`, `shot NAME`).
  It posts clicks to A-Tuning's Vcore buttons and Apply, then captures the window to confirm the value. One press is 5 mV.
- **Step script**: wait for water at or below a start temperature and an idle GPU (optional), set the voltage, verify the capture, run 10 minutes with GPU heat.
- **Sweep loop**: step down 10 mV at a time, back to back, stop at the first non-pass and set the last passing value back.
- **Overnight loop**: 3 x 30 minutes per voltage, move up 10 mV on any non-pass, stop at the first voltage with 90 clean minutes, with a ceiling.
- **Journal**: a line is written before every step, so the last line names what was running if the PC locks up.

## RAM sweep scripts

- **Sweep controller** (Python, runs as a systemd user service in WSL at every boot): builds the BIOS script, writes it, reboots,
  checks for crashes since the step was set, verifies the effective values, classifies PASS / FAIL / CRASH / NO TRAIN / NOT APPLIED,
  runs the test twice, and keeps state in a JSON file and a Markdown journal.
  NO POST and FREEZE cannot be detected by a controller that is not running. A person marks those afterwards.
- **Groups and gates**: steps are grouped, a group stops at its first failure, and an "alternatives" group tries the next option instead.
  Groups marked risky wait until the owner has said he is around. CPU-side groups add a Cinebench run with GPU heat through Shakedown.
- **NVRAM helper**: parses SCEWIN exports and rewrites chosen questions by token.
- **SCEWIN task**: an elevated scheduled task that runs an export or an import of a staged file. The BIOS admin password stays in an admin-only file and never appears in the scripts or the journal.
- **Failsafe task**: runs as SYSTEM at startup. If a step's boot repeats without its test starting, it restores the last good settings and reboots once.
  Version 2 (14 Nov) takes a fresh export at that moment and writes the wanted numbers into it, because a prepared script does nothing on fallback timings.
- **Step cards**: before every risky reboot a short card goes to the other machine: what is set, what the owner may see, and the one BIOS field (with its exact number) that undoes it.
- **Memory track** (14 Nov on): runs the long memory checks without reboots or BIOS writes. A temperature guard reads every core every 20 s and ends the load at 100 C on the hottest core.
  It also checks that cores outside the allowed set are idle, and refuses to start without a DIMM temperature reading.
- **memhold**: our at-rest test. Fills N GiB with patterns, locks it in physical memory, leaves it, and re-checks it every 2 minutes.
  A mismatch is re-read after a cache flush, so a wrong value stored in the DIMM is told apart from a bad read. Proven with an injected bit before use.
- **TestMem5 runner**: starts TM5 with a fixed config, confines it once at launch, reads its log, and reports PASS, FAIL or "not a result".
- **memlat**: our pointer-chase latency and read bandwidth test (a few dozen lines of C). It uses 4 KB pages.
  Its numbers are for before and after on one machine. They are not AIDA64 numbers.
- **dimmtemp**: reads both DIMMs' SPD hub temperatures over SMBus through PawnIO's SmbusI801 module (read only).

## Third-party tools

| Tool | Use |
|---|---|
| [Cinebench 2026](https://www.maxon.net/en/tech-info-cinebench) (standalone) | CPU load. Command-line flags let you set a minimum duration and read the score from the console |
| [y-cruncher](https://www.numberworld.org/y-cruncher/) 0.8.7 | Memory and IMC stress (FFTv4, N63, VT3), plus in-cache tests to rule out cores |
| [PawnIO](https://pawnio.eu) and [PawnIO.Modules](https://github.com/namazso/PawnIO.Modules) | Signed driver for MSR and SMBus reads |
| SCEWIN (AMI) | BIOS setup export and import. See [SCEHUB](https://github.com/ab3lkaizen/SCEHUB) |
| HWiNFO | Voltages (including DIMM PMIC VDD and VDDQ), DIMM sensor initialisation, cross-checks. It can take the Super I/O chip away from a fan controller while it is open |
| ASRock A-Tuning | Live ratio and Vcore on this board |
| [CoreCycler](https://github.com/sp00n/corecycler) | Per-core cycling. Not used yet; recommended for light-load errors |
| [TestMem5](https://github.com/CoolCmd/TestMem5) 0.13.1 | DRAM pattern test, anta777 config. It found the heat-bound tREFI fault that y-cruncher missed. This build reads its settings from TM5.ini and writes Log.txt, so it can run unattended |

## Tooling pitfalls we hit, so you do not have to

- **y-cruncher.exe is only a launcher.** The work runs in a child process. Set core affinity on that worker, not on the launcher, and hold it for about 20 s after start: a mask set in the first instant is undone by y-cruncher's own start-up. Check by reading per-core load. Ending the launcher can leave the worker running.
- **Confine TestMem5 once, at launch. Never move it mid-run.** It runs as 32 worker processes.
- **TestMem5 0.13.1 writes only "Testing completed" on a clean pass.** Our parser expected other wording and logged a clean 60-minute run as FAIL.
  Count "An error N occurred in the program" as a failure: on unstable settings it is most likely memory corruption.
- **TestMem5 coverage drops if it starts right after another memory test,** because the memory is not yet free. Check the "Testing N MB x 32 (P%)" line. Ours fell to 40%. Wait a minute and restart if it is far below about 75%.
- **Always write the full BIOS set.** Our writer copied a golden script over every field it was not given. A write of three fields silently put every other timing back to XMP. A read-back caught it before the reboot.
- **Read back stored values after every write, and live values after every boot.**
- **Short single runs to search, long runs only at checkpoints.** 60-minute runs for every probe wasted hours.
- **A temperature guard on every load,** and test the guard at low load first. Ours let an "E-cores only" load run on all 32 threads at 430 W for 100 s, twice, before the affinity fix.
- **Mark planned power-offs,** so a repaste does not show up as a crash.
- **Close GPU-heavy programs before a scripted restart.** A large model server being torn down mid-shutdown hung Windows (0x9F) for about 5 minutes each time.
- **Flush your journal to disk.** A reboot once left ours ending in zero bytes.
- **Two programs on one HID device see each other's replies.** Windows copies every input report to every open handle. Flush the input queue before each request and reopen the handle on a reply that does not match. See [cpu/quirks.md](cpu/quirks.md).
- **WSL2 keeps Linux's file cache in Windows memory.** Task Manager showed VmmemWSL at 23 GB while Linux programs used 1.7 GB; the rest was page cache from the test files.
  `wsl.exe -u root sh -c "echo 3 > /proc/sys/vm/drop_caches"` gave it back. `autoMemoryReclaim=dropCache` under `[experimental]` in `.wslconfig` makes it automatic. Free the memory before a memory test sizes itself.
- **Undo the sweep settings at the end.** We had turned Windows Recovery off, set the boot status policy to ignore failures and armed a failsafe task. Keep a list, and put every item back when the campaign ends.

## Reproduce it on your machine

CPU:

1. Read your event log first. Count WHEA event 19 records by APIC ID. That is your free per-core map.
2. Run your all-core load and see which cores fail. Hold them out and run again. Repeat until it passes.
3. If your board can, set per-core ratios. If not, find the all-core ratio the weakest core holds.
4. Heat the loop the way you actually use it (GPU load on a shared loop). Start each run warm.
5. Find the voltage floor with short runs, back to back. Then validate 3 x 30 minutes hot. Add margin. Near the floor, run every step for 30 minutes: two of our configurations passed 10 minutes and failed 30.
6. Count silent crashes as fails. Keep a ledger of every run, including the ones you would rather forget.

RAM:

1. Save your working profile to a BIOS slot. Find the Clear CMOS jumper.
2. Decide how you will recover before you start. A failsafe task covers crash loops and training fallbacks.
   A no-POST or a hard freeze needs you at the power button, or remote power control. We had both in one day.
3. Turn crash dumps off and disable Windows Recovery for the sweep. Write down everything you changed so you can undo it.
4. Log DIMM temperature on every test. Note the temperature at the first error. Compare steady peaks, not cold starts.
5. Bring TX and VDD2 to daily-safe values first. Then turnarounds through tWRRD, then tRFCpb, tRAS, tCWL, then tREFI in stages.
   Heat-limited items and anything that can stop training come last, one clock at a time.
6. Two passing runs before a value is kept. Run a control whenever failures cluster.
7. Read the effective values back after every boot, including the ones you did not touch. Watch for the fallback signature.
   Diff the whole timing page against your baseline now and then. Ours showed tWTR had moved and tRTP had not.
8. Give ring and E-core changes a CPU load test as well as the memory test.
9. Write a prediction for each risky step, then test anyway. See [ram/predictions.md](ram/predictions.md).
10. After every unclean stop: `chkdsk /scan`, `sfc /scannow`, and an export to confirm what the BIOS is running.
11. Finish with long runs: an at-rest test, 60 minutes of y-cruncher and 60 minutes of TestMem5, at the DIMM temperature real use reaches, with the CPU load kept cool.
12. Then, separately, the CPU with the GPU loaded, and a few hot reboots.
13. Keep small crash dumps on, so a slow boot or a reset has a stop code you can read.

If you automate BIOS writes, test your recovery path with no change first. Ours had a bug in it.
