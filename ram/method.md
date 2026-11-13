# RAM method

The owner wanted RAM tuning to run on its own, from inside the OS, without someone at the BIOS for every step.
Timings need a retrain, so every step is a BIOS write plus a reboot. Here is how we made that unattended and recoverable.

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
| Windows boot settings | `bcdedit` recovery disabled and boot failures ignored, so a crash loop retries instead of stopping in Windows Recovery |
| Crash dumps off during the sweep | A bugcheck reboots straight away, instead of hanging while writing a dump from unstable RAM |
| Boot Failure Guard (BIOS) | After 4 failed boots the BIOS loads defaults |
| An external watcher | Another machine alerts if this one stays unreachable for 3 minutes outside expected reboot windows |

The failsafe runs about 18 s into a boot. A step that crashes earlier than that on every boot still needs a person (or Boot Failure Guard).
We tested the failsafe on purpose with unchanged settings before relying on it. The first test found a script error in the failsafe itself.

Also paused for the sweep: background services and the cloud sync client, to keep the test conditions the same between steps.

## 3. Per-step test

1. Build the script: the pre-sweep BIOS export + every value kept so far + this step's change.
2. Write it with SCEWIN. Write "testing X" to the journal. Reboot.
3. After boot, read the effective values from a fresh export. Classify:
   - values as written: run the test,
   - the BIOS changed our value but nothing else: **NOT APPLIED**,
   - values we did not touch changed too (fallback timings): **NO TRAIN**.
   Boot time is measured too. A trained boot takes about 45 s. A failed training took 135 to 282 s.
4. Measure latency and bandwidth (`memlat`: our own pointer-chase latency plus read bandwidth).
5. Stress: y-cruncher 0.8.7, tests FFTv4, N63 and VT3, 7 minutes, 30 GB.
6. Check the WHEA log for new records.
7. Log DIMM temperatures through the run (peak and end, both DIMMs).
8. Since plan v2: **two passing runs in the same boot** before a value is kept.

Outcome types used in [results.md](results.md):

| Outcome | Meaning |
|---|---|
| PASS | Booted with our values, test clean, WHEA 0 |
| FAIL | Booted with our values, the test found an error |
| CRASH | Windows crashed at boot, login or during the test |
| NO TRAIN | Memory training failed, the BIOS fell back to safe timings |
| NOT APPLIED | The BIOS replaced our value with its own and booted |

## 4. DIMM temperature logging

- Each DDR5 DIMM has an SPD hub with a thermal sensor. We read it over SMBus through PawnIO's signed SmbusI801 module (read only, byte reads).
- After a cold boot both sensors read 0 until something initialises the SPD hubs. HWiNFO does. So each test starts HWiNFO briefly if the reading is 0.
- HWiNFO and our reader agreed within a degree.

## 5. Control runs

A failed step proves the step failed only if the base still passes.
When three steps in a row failed late in the test, we stopped and ran the kept settings with no change.
That control failed too, so the base was marginal and those three FAILs were not proof against their values.
We then stepped back the most recent additions one at a time until the control passed twice. See [results.md](results.md).

Rule: if you see several late FAILs in a row, run a control before you believe any of them.

## 6. Why order matters

Our first plan went tRFC, tWR, tRRD_L, tWTR, tRCD/tRP, tCL, tRAS, tREFI. Problems with that:

- **Voltages were last, at XMP's 1.45 V on TX and VDD2.** That is above every daily recommendation we found. Plan v2 lowers TX and VDD2 first.
- **tCL 38 went in before tRAS and tREFI.** It was marginal, so it poisoned every later step until a control found it.
- **Changing tCL or tCWL shifts tWR and tWTR** through tWRPRE and tWRRD. Secondaries tuned before primaries can move.
- **Temperature-sensitive timings (tREFI, tCL, the last tRFC steps) need cool DIMMs.** Ours peaked at 54 to 66 C. Those steps wait for the water block.

Plan v2 order: (1) TX and VDD2 down to daily-safe values; (2) tWRRD_dg/sg (which control tWTR), tRDWR, tRDRD_sg, tWRWR_sg;
(3) tRFCpb and tRAS. After the water block: tCL 38 with tCWL 38, tRFC about 651, tREFI, tRCD 49/48.
Final validation: VT3 for 60 minutes, TM5, cold-boot retrains, a run with GPU heat, before/after benchmarks.

## 7. After a crash

- `chkdsk /scan C:` (online, read only) and `sfc /scannow`.
- Check the Linux file systems if you run WSL.
- Check the BIOS did not fall back to defaults before the next step (Boot Failure Guard).
