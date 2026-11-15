# RAM method

The owner wanted RAM tuning to run on its own, from inside the OS, without someone at the BIOS for every step.
Timings need a retrain, so every step is a BIOS write plus a reboot. Here is how we made that unattended and recoverable, and where it stopped being either.

## 1. Writing BIOS settings from Windows: SCEWIN

- SCEWIN is AMI's setup-variable tool. It exports every BIOS setup question to a text file and imports changes.
  We used version 5.05.01.0002 from a board vendor's official utility package. All three files are AMI-signed.
- **Export works out of the box**, even with VBS and HVCI on.
- **Import is blocked on ASRock Z590 and newer** by "Password protection of Runtime Variables"
  ([SCEHUB](https://github.com/ab3lkaizen/SCEHUB)). SCEHUB says you need a modded BIOS to turn it off.
- **The workaround that worked here:** set a BIOS Administrator (supervisor) password, then import with `/cpwd`
  ([recipe](https://gist.github.com/Arie/af8e353dec6ce6b47359529571ace45f)):

```
SCEWIN_64.exe /i /lang en-US /ms "Realtime Memory Timing" /qv 0x1 /cpwd <your BIOS admin password>
```

  Output: "Password accepted ... imported successfully ... Variable access re-locked". A re-export showed the new value, and it survived a reboot.
- Keep the password in a file only administrators can read. Run SCEWIN from an elevated scheduled task, so the test controller never needs a UAC prompt.
- **SCEWIN matches a question by its full prompt, and the prompt includes the live value.** A script built from an old export silently skips changed questions
  ("unmatched question"). Always build the import script from a fresh export, then copy your values in.

Risk: writing the wrong variable can stop the board booting. This board has a Clear CMOS jumper and no BIOS flashback. Change only questions you can name.

## 2. Unattended reboots

| Piece | What it does |
|---|---|
| Windows autologon | Logs in after each reboot. A logon task locks the workstation at once if uptime is under 3 minutes |
| Linux user services (WSL, systemd, linger) | The sweep controller starts at boot by itself |
| Failsafe startup task (SYSTEM, at startup) | If a step's boot repeats without its test starting, it imports the last good settings and reboots |
| Windows recovery disabled | `bcdedit` with `recoveryenabled No` and `bootstatuspolicy IgnoreAllFailures`, so a crash loop retries instead of stopping in Windows Recovery |
| Crash dumps off during the sweep | A bugcheck reboots straight away, instead of hanging while writing a dump from unstable RAM. Later turned back to small dumps: with dumps off we misread shutdown hangs as slow training for two days |
| Crash check on every step | Looks for unclean-shutdown events since the step was set. A hit means CRASH, and the last good settings go back |
| Boot Failure Guard (BIOS) | After 4 failed boots the BIOS steps in. On this board it did not load defaults. See section 8 |
| An external watcher | Another machine alerts if this one stays unreachable for 3 minutes outside expected reboot windows |

We tested the failsafe on purpose with unchanged settings before relying on it. The first test found a script error in the failsafe itself.

Also paused for the sweep: background services and the cloud sync client, to keep the test conditions the same between steps.
All of this has to be undone afterwards: crash dumps back on, Windows recovery back on, the failsafe task off, services back.
Keep a restore list as you go.

## 3. Per-step test

1. Build the script: the pre-sweep BIOS export + every value kept so far + this step's change.
2. Write it with SCEWIN. Arm the failsafe. Write "testing X" to the journal. Reboot.
3. After boot, check for crashes since the step was set. Then read the effective values from a fresh export. Classify:
   - values as written: run the test,
   - the BIOS changed our value but nothing else: **NOT APPLIED**,
   - values we did not touch changed too (fallback timings): **NO TRAIN**.
   Boot time is measured too. A trained boot takes about 45 s, a reboot with no BIOS change about 20 s.
   A long boot with the fallback values is a failed training. A long boot with your own values is probably a shutdown hang: check for a 0x9F record before you conclude anything.
4. Measure latency and bandwidth (`memlat`: our own pointer-chase latency plus read bandwidth).
5. Stress: y-cruncher 0.8.7, tests FFTv4, N63 and VT3, 7 minutes, 30 GB.
6. Check the WHEA log for new records.
7. Log DIMM temperatures through the run (peak and end, both DIMMs).
8. Since plan v2: **two passing runs in the same boot** before a value is kept.
9. For CPU-side steps (ring, E-cores): also 10 minutes of Cinebench with GPU heat. Ring 51 passed steps 4 to 8 and failed this one.

Outcome types used in [results.md](results.md):

| Outcome | Meaning | Who recovers |
|---|---|---|
| PASS | Booted with our values, both runs clean, WHEA 0 | |
| FAIL (test) | Booted with our values, a test found an error or the load crashed | the sweep |
| CRASH | Windows crashed at boot, at login or during the test, and restarted | the sweep or the failsafe, if Windows gets far enough |
| NO TRAIN | Memory training failed. The BIOS fell back to safe timings and booted (signature: tRFC 395) | the sweep |
| NOT APPLIED | The BIOS replaced our value with its own and booted at normal speed | nothing to recover |
| NO POST | No picture at all. The board sat powered and never reached the BIOS screen | **a person** |
| FREEZE | The running system stopped dead. No bugcheck, no restart | **a person** |

## 4. DIMM temperature logging

- Each DDR5 DIMM has an SPD hub with a thermal sensor. We read it over SMBus through PawnIO's signed SmbusI801 module (read only, byte reads).
- After a cold boot both sensors read 0 until something initialises the SPD hubs. HWiNFO does. So each test starts HWiNFO briefly if the reading is 0.
- HWiNFO and our reader agreed within a degree.
- Compare steady peaks only. Our first reading with the new DIMM fan was taken from a cold start and overstated the fan by about 5 C.

## 5. Control runs

A failed step proves the step failed only if the base still passes.
When three steps in a row failed late in the test, we stopped and ran the kept settings with no change.

| Control | Settings | Result |
|---|---|---|
| A | the kept set as it was (tCL 38, tRAS 76) | FAIL |
| B | A with tRAS back to 96 | FAIL (and an in-cache run with no memory traffic passed, so it was memory, not cores) |
| C | B with tCL back to 40 | PASS, twice |

So tCL 38 at DIMM 1.45 V was the marginal setting. It had passed once.
The three FAILs measured on top of it (tRAS 64, tREFI 65535, tRFC 670) were not proof against those values.
Two of the three later passed on a stable base. The third, tRAS 64, was below the real tRAS floor anyway.

Rule: if you see several late FAILs in a row, run a control before you believe any of them.

## 6. Plan v1 and why it was wrong

Our first plan went tRFC, tWR, tRRD_L, tWTR, tRCD/tRP, tCL, tRAS, tREFI, one run per step. Problems with that:

- **Voltages were last, at XMP's 1.45 V on TX and VDD2.** That is above every daily recommendation we found for TX.
- **tCL 38 went in before tRAS and tREFI.** It was marginal, so it poisoned every later step until a control found it.
- **One run per step.** tCL 38 passed its single run.
- **Changing tCL or tCWL shifts tWTR** through tWRRD. Secondaries tuned before primaries can move.
- **Big first steps on unknown ground.** tRCD/tRP went 52, 50, 48 and tRRD_L went 12 to 10. Both ends crashed Windows at boot.

## 7. Plan v2: the order, and why

Plan v2 started from the base that passed control C twice: tRFC 680, tWR 24, tRCD/tRP 50, tCL 40, tRAS 96.
Every step needs two passing runs in the same boot.

| # | Group | Why here |
|---|---|---|
| 1 | Voltages first: TX 1.45 down to 1.40, 1.35, 1.30 V; VDD2 1.45 down to 1.42, 1.40 V | Get every rail into the range experienced tuners run daily before judging any timing. Less voltage also means less heat |
| 2 | Tertiaries through tWRRD: tWRRD_dg, tWRRD_sg, then tRDWR_dg/sg, tRDRD_sg | On Intel, tWTR is set through tWRRD. The _dg timings carry most of the gain |
| 3 | tRFCpb | A refresh timing the board left at the spec value. Soft failure expected if it fails |
| 4 | tRAS | Cheap, low risk. The floor is tRCD + tRTP |
| 5 | tCWL (38 at tCL 40) | Matches the DRAM's own CL minus 2. No voltage cost |
| 6 | tREFI, in steps (40960, 49152, later 57344, 65535) | Largest refresh gain. Only after tRFC is settled, and it is the most heat-sensitive |
| 7 | CPU extras: ring 51, E-cores 46 | Separate from RAM changes, with a Cinebench run on top |
| 8 | Heat-limited items: tRFCpb 440 and 420, tREFI 57344 and 65535, tCL 38 with tCWL 36 (then with DIMM VDD/VDDQ 1.50/1.48 V), tRFC 670 and 660 | First planned for after the water block. The owner asked to map them with the DIMM fan instead |
| 9 | Risky items, with the owner near the PC: ring 51 at 1.350 V, tRCD/tRP 49, tRRD_L 11 | Steps whose neighbours crashed at boot, or that change the daily Vcore. One clock at a time |
| 10 | Leftovers: tWR 22 and 20, tRTP, tRAS 72 and 68 | Small gains, expected to fail softly if they fail |
| 11 | Retries of the old crashes: tRCD/tRP 48, tRRD_L 10 | Only once 49 and 11 had passed |

E-cores 47 ran in group 8, unattended, as a step we expected to fail softly. It froze the PC instead. See section 8.

Rules that came with it:

- **One clock at a time near a known failure.** The old crashes came from two-clock jumps. The single steps before them passed.
- **Steps that can stop training wait for a person.** After the no-POST, the sweep ran those only while the owner had said he was around.
- **Raise a CPU-side memory voltage only for a new training failure.** After the no-POST we put TX back from 1.30 to 1.35 V for training margin. Both are inside the daily range.
- **Leave tRDRD_sg and tWRWR_sg trained.** Little gain, heat-sensitive, and the one step that gave us a no-POST. (Reversed on 14 and 15 November: both went to 15, one clock each, with the owner around.)
- **Read back coupled timings after every change.** See the tWTR, tRTP and tRAS entries in [quirks.md](quirks.md). We got all three wrong at first.

A two-run pass in a sweep finds candidates. It does not make a daily profile. What the long checks found is in sections 10 and 11.

## 8. What unattended operation cannot do

Two failures on 13 November needed hands. The failsafe could not help with either, by design.

**No POST (tRDRD_sg 14).** After the reboot the PC stayed powered with no picture on any monitor. No POST, no BIOS screen.

- The failsafe is a Windows startup task. Windows never started, so it never ran.
- The board did not retrain by itself while it sat there.
- The owner power-cycled it by hand five times. After the fifth, Boot Failure Guard showed its message and continued.
- **Boot Failure Guard did not load defaults.** A SCEWIN export showed NVRAM still held every setting, including the bad one.
  Only the memory had come up in a fail-safe mode: DDR5-3600, tCL/tRCD 29, tRFC 395, TX 1.100 V, VDD2 1.120 V.
- That was lucky for us: one write of the last good set and one reboot restored everything. But do not count on Boot Failure Guard to clean up.

**Hard freeze (E-cores 47).** About 3 minutes into the stress test the system stopped dead. The lock screen clock stood still.

- No bugcheck, so no automatic restart. Crash dumps off and auto-restart on make no difference to a freeze.
- The external watcher alerted. It could not press the button.
- The owner held the power button. Windows logged only the unclean shutdown.

What follows from that:

- A startup failsafe covers crash loops and soft training fallbacks. It does not cover a no-POST or a hard freeze. Only a person or remote power control does.
- Run steps that can do either only when someone can reach the power button. Mark them in the plan.
- Turn crash dumps off during a sweep. With dumps on, one blue screen at boot sat frozen for about an hour writing nothing.
- Disable Windows Recovery during a sweep, or a crash loop ends at a recovery screen waiting for a click.
- The failsafe runs about 18 s into a boot. A step that crashes earlier than that on every boot also needs a person.

## 9. After every unclean stop

- `chkdsk /scan C:` (online, read only) and `sfc /scannow`.
- Check the Linux file systems if you run WSL.
- Confirm by export that the BIOS runs the last good set before the next step.

We ran these after every crash, the no-POST and the freeze. They were clean each time.
The only repair was on the very first `sfc` run: 62 Start Menu shortcut files, no system binaries, nothing unrepairable.
That is a relief, not a guarantee. Unstable RAM can corrupt data that no checker looks at.

## 10. What changed in the sweep on 13 and 14 November

Faults found in our own sweep, and the fixes:

- **After a FAIL the next step was written from the failing session,** on top of the failing values. That is how tCL 36 went in straight after tWRRD_sg 62 had failed.
  Now every failure gets: restore, reboot, confirm by export, and only then the next step.
- **The failsafe could not restore from fallback timings** (see [quirks.md](quirks.md)). Failsafe v2 takes a fresh export at that moment, writes the wanted numbers into it, flushes its marker and log to disk, and counts starts instead of comparing names.
  It was tested live: a wrong value stored on purpose was put back unattended.
- **It still cannot help** when Windows dies before the task has run twice. tRCD/tRP 48 and tCL 36 did exactly that. Those steps need a person.
- **A step card before every risky reboot.** It goes to the other machine and says what was set, what the owner may see, and the one BIOS field that undoes it.
  A bug in the cards once said "Vcore back to Auto", which on this board means adaptive voltage far above the owner's ceiling. Cards now name the exact number.
- **The slow-boot rule measured the wrong thing.** It treated a long boot as a slow training. The long boots were shutdown hangs. Every verdict built on boot time was voided and re-tested.

## 11. Long checks, and how we run them now

The first long validation (14 November) crashed 25 minutes into a 60-minute y-cruncher run, and our script started the 60 minutes again without noticing.
Later that day an at-rest memory test with GPU heat beside it hard-reset the PC at 459 W. Both taught us something about the method:

1. **Keep the CPU out of a RAM test.** An all-thread memory test with the GPU heating the loop was the heaviest CPU load of the whole campaign. A crash there tells you nothing about the RAM.
   Now: no GPU heat, the load confined to the E-cores and the cooler P-cores, and a guard that ends the load at 100 C on the hottest core (twice in a row, or 104 C once).
2. **Three kinds of memory test, because they find different faults:**
   - an **at-rest test**: fill 30 GiB with patterns, lock it in physical memory, leave it untouched, and check it every 2 minutes for 45 minutes, with y-cruncher in 8 GB beside it. A flip in held data points straight at refresh. The re-read flushes the cache line first, so a wrong value really stored is told apart from a bad read;
   - **y-cruncher** VT3 / FFTv4 / N63, 60 minutes;
   - **TestMem5 0.13.1**, anta777 config, 60 minutes. This one found the tREFI fault the other two missed.
3. **A failed test gives back one setting and repeats only that test.** Passed phases are not re-run after loosening.
4. **Short single runs to search, long runs at checkpoints.** We once stacked about 25 kept changes on 16-minute tests, so the first long run implicated all of them.
   On 15 November the last eight probes ran one 8-minute run each, and the final validation confirmed the lot.
5. **Count every hardware record over the whole validation,** not only inside each test window.
6. **Write a pre-registered reading** before a test whose result you want to interpret, so you cannot fit the story afterwards.

Final validation on 15 November: at-rest 45 minutes, y-cruncher 60 minutes, TestMem5 60 minutes, all clean. See [results.md](results.md).
Cinebench 30 minutes with GPU heat and five hot reboots passed on 14 November, on the set of that day (tREFI 65535, tRP 48, tWR 16).
