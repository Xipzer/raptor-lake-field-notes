# SCEWIN: writing BIOS settings from Windows

Everything we learned driving AMI SCEWIN (AMISCE) on an ASRock Z790 board, BIOS 10.01, through every step of an unattended RAM sweep (one BIOS write and one reboot per step).
The same tool runs on most AMI-based boards (MSI, ASRock, ASUS, Gigabyte and many OEMs), so most of this carries over. Board-specific items are marked.

Labels: **Measured** (seen on our machine), **Source** (a public source says so, link given), **Estimate** (our reasoning).

## Quick reference

| Quirk | What you see | Remedy |
|---|---|---|
| [Import is password-locked on ASRock](#1-import-is-locked-on-asrock-z590-and-newer) | "Error in writing variable", "System configuration not modified" | Set a BIOS Administrator password, import with `/cpwd` |
| [Prompts embed the live value](#2-every-prompt-contains-the-live-value) | A changed setting is silently skipped | Build every import from a fresh export |
| [Partial writes](#3-write-the-full-set-every-time) | A three-field write puts every other timing back | Always write the whole intended set, read back before rebooting |
| [TcgStorageDynamicSetupVar](#4-harmless-noise) | An error on every import | Ignore it if your own fields read back right |
| [Labels are not values](#5-value-label-and-what-the-bios-really-runs) | The stored value and the trained value differ | Read the label for live values; some fields have none |
| [Read-back offsets](#6-read-back-offsets-and-derived-fields) | tRTP reads one lower, tWTR moves by itself | Expect the offset; diff the whole page |
| [Duplicate questions](#7-duplicate-questions) | One name, two blocks | Only edit tokens that appear once |
| [Fallback timings change prompts](#8-after-a-training-fallback-the-prompts-change) | A prepared restore script does nothing | Take a fresh export at restore time |
| [Saved OC profiles](#9-what-scewin-cannot-reach) | Not in the export | Save and load profiles in the BIOS by hand |
| [Result file race](#10-tooling-races) | A half-written result read as a failure | Wait for an explicit end marker |

## 1. Import is locked on ASRock Z590 and newer

- **Measured:** export works out of the box, even with VBS and HVCI on. Import fails.
- **Source:** ASRock (Z590+, B560+, X670+ and newer) protect runtime variables. SCEHUB lists the symptom ("Warning: Error in writing variable" and "System configuration not modified") and says a modded BIOS is needed to turn the protection off ([SCEHUB](https://github.com/ab3lkaizen/SCEHUB)).
- **What worked here:** set a BIOS **Administrator (supervisor) password**, then pass it to SCEWIN with `/cpwd` ([recipe gist](https://gist.github.com/Arie/af8e353dec6ce6b47359529571ace45f)).
  The output then reads `Password accepted ... imported successfully ... Variable access re-locked`. No modded BIOS needed.
- **Side effect:** entering the BIOS now asks for that password. Write it down somewhere safe.
- Keep the password in a file only administrators can read, and run SCEWIN from an elevated scheduled task. Your scripts then never hold the password, and the controller never triggers a UAC prompt.
- We got SCEWIN 5.05.01.0002 from a board vendor's official utility package. The three files (`SCEWIN_64.exe`, `amifldrv64.sys`, `amigendrv64.sys`) are AMI-signed. Check the signature on whatever copy you use.

## 2. Every prompt contains the live value

This is the big one.

- An export lists each setup question with its prompt. On this BIOS the prompt of many DRAM fields includes the current value, for example `tRFC 922`.
- **SCEWIN matches a question by its full prompt.** A script made from an older export carries the old prompt text. After the value has changed, the prompt no longer matches, and SCEWIN skips the question.
- **Measured:** our tRFC 740 step was silently skipped (12 November): the script came from an export taken before tRFC 820 was applied. The import reported success for everything else.
- **Remedy:** never keep "template" scripts. Before every write: take a fresh export, change only the values you want in that text, write it back, then re-export and compare.
- **Public sources:** SCEHUB documents "Unmatched question" and "System configuration not modified" but gives other causes (no `*` on an option, unchanged script). AMI's own release notes list "SCE question mismatch when prompt has trailing space" ([version history mirror](http://forum.ru-board.com/topic.cgi?forum=81&limit=1&m=1&start=1062&topic=0275)). We found **no public report** that a live value inside the prompt breaks matching. That part is our own finding.

## 3. Write the full set every time

- **Measured (14 November):** one of our writers took a "golden" export and applied only the fields it was given. A write of three fields therefore put every other tuned timing back to its XMP value. The read-back of the stored values caught it before the reboot.
- **Remedy:** pass the complete intended set (every field you have tuned, not only the change). Read back from a fresh export and require zero mismatches before you reboot. Then confirm with another export after the boot.
- Nothing public found on this; it is a property of how you build scripts, not of SCEWIN.

## 4. Harmless noise

- **Measured:** every import printed `Error in writing variable TcgStorageDynamicSetupVar to NVRAM`. Every one of our own fields still applied.
- The output also starts with PowerShell `NativeCommandError` noise when SCEWIN is run from a script, because SCEWIN writes to stderr.
- How we judged success instead: `Password accepted` plus `imported successfully` in the output, and a clean read-back.
- `System configuration not modified` means nothing changed. That is fine for a no-op, and a red flag if you expected a change (see section 2).
- **Source:** SCEHUB lists "Error in writing variable" as a write-protection symptom. We found **no public report** of this particular variable being harmless; treat it as harmless only if your read-back is clean.

## 5. Value, label, and what the BIOS really runs

- Many DRAM questions have two parts in the export: the stored **value** (what you asked for, 0 = Auto) and a **label** in the prompt (what training actually produced).
- After a boot, compare labels, not values, to see what the board runs.
- **Fields with no label cannot be proven live from an export.** On this board: Vcore, the E-core ratio, DIMM VDD and VDDQ, and the per-core ratios. We verified them elsewhere: DIMM PMIC readings in HWiNFO (1.500 / 1.485 V for 1.50 / 1.48 V set), and the measured E-core clock (4,580 MHz for 46x).
- **Measured:** ASRock VDD_IMC is VDDQ_TX and VDD_CPU is VDD2 (read in HWiNFO under load: 1.350 V and 1.448 V for 1.350 and 1.450 V set).

## 6. Read-back offsets and derived fields

| Field | Behaviour on this BIOS (Measured) |
|---|---|
| tRTP | Reads back as the entered value **minus 1**. Entered 13 reads 12 (the DRAM minimum). Not derived from tWR |
| tWTR_S / tWTR_L | Follow tWRRD_dg / tWRRD_sg: tWTR = tWRRD - tCWL - 6. A direct write of 36 / 12 came back 42 / 18 |
| tCL in Gear 2 | Must be even |
| tRAS | Below tRCD + tRTP it is no longer the binding limit. Not a hard floor here |

- **Source:** the 6-cycle tWTR offset is reported by other ASRock owners (overclock.net Intel DDR5 and Lightning threads). Pakhtunov's guide gives a different constant (tWRRD - tCWL - 10, [formulas](https://pakt.cc/en/ddr5/formulas/)). Use what your own board reads back.
- An automated checker must expect these offsets. Ours first classed tRTP read-backs as NOT APPLIED and skipped a whole family of steps for hours.
- Diff the **whole** DRAM page against the baseline now and then, not only the fields you touched. That is how we found tWTR had moved for eleven hours without us.

## 7. Duplicate questions

- Some questions appear twice in an export, with a note like "Only one question will be enabled".
- Our editor only touches tokens that occur exactly once and refuses the rest. Edit duplicates in the BIOS by hand if you must.
- Keep line endings and encoding byte-exact. Lines can carry stray ANSI colour fragments; leave them alone.

## 8. After a training fallback the prompts change

- When DRAM training fails, this board boots at fail-safe settings: DDR5-3600, tCL / tRCD / tRP 29, tRFC 395, tWR 54, TX 1.100 V, VDD2 1.120 V, tREFI 3510.
  Any untouched field changing is the signature of a failed training (NO TRAIN), as opposed to the BIOS simply refusing one value (NOT APPLIED).
- Because the prompts embed live values, every prompt now differs from your prepared "last good" script. **Our first startup failsafe imported a prepared script, and on fallback timings it did nothing** ("System configuration not modified").
- **Remedy (failsafe v2):** at restore time take a fresh export and write the wanted numbers into it, field by field, then import. See [the failsafe section](#the-startup-failsafe).

## 9. What SCEWIN cannot reach

- **Saved UEFI OC profiles** are not in the export. Save, load and delete them in the BIOS yourself.
- Settings that only apply after a full power cycle still need one.
- Everything applies at the next restart. Nothing a SCEWIN import does is live, except fields the BIOS exposes as runtime-capable (we did not test "Realtime Memory Timing" live under VBS).
- **Boot Failure Guard** (Enabled, count 4) did not load defaults after our no-POST: NVRAM still held the bad value. Do not count on it to clean up.

## 10. Tooling races

- Our SCEWIN task writes its output to a result file. Reading that file before SCEWIN finished gave us a false "could not read the BIOS" once.
- **Remedy:** the task writes an explicit `done HH:mm:ss` line last. The reader waits for that line.

## The token map approach

SCEWIN names questions by text, and the text can change (section 2). The **token** (a hex id per question) does not. So we keyed everything on tokens:

1. Take a full export once (ours: 5,414 questions, 1.6 MB) and find the tokens for the fields you care about, by name.
2. Keep a small table: token, field, unit, notes (read-back offsets, no-label fields).
3. A parser reads an export into blocks by token and offers `value(token)`, `label(token)`, `set(token, value)` and `text()` that writes the script back byte-exact.
4. Every write: fresh export, `set` each wanted token, write, import, re-export, compare by token.

Our map for ASRock Z790I Lightning WiFi, BIOS 10.01 (tokens may differ on other boards or BIOS versions; check yours):

| Token | Field | Final value |
|---|---|---|
| 2C4A / 2C4B | P-core ratio mode / all-core ratio | All Core / 57 |
| 2C6E..2C75 | Per-core max ratio, core 0..7 | unused |
| 2C5D | E-core ratio | 46 |
| 2C77 | CPU Cache (ring) ratio | 50 |
| 2CEE / 2CEF | Vcore mode / fixed Vcore in mV | Fixed / 1320 |
| 2C9F | Load XMP setting | XMP 3.0 Profile 1 |
| 2CA5 | tCL | 38 |
| 2CB5 | tCWL | 34 |
| 2CA7 / 6AA | tRCD / tRCDW | 49 / 32 |
| 2CA8 / 2CA9 | tRP / tRAS | 46 / 64 |
| 2CB3 | tRTP (reads entered minus 1) | 13 entered, 12 live |
| 2CAB / 2CAF | tWR / tRRD_L | 12 / 11 |
| 2CAC / 2CAE / 2CAD | tRFC / tRFC2 / tRFCpb | 660 / 660 / 420 |
| 2CB6 | tREFI | 32767 |
| 2CD2 / 2CD1 | tWRRD_dg / tWRRD_sg | 46 / 64 |
| 2CCE / 2CCD | tRDWR_dg / tRDWR_sg | 19 / 19 |
| 2CC9 / 2CD5 | tRDRD_sg / tWRWR_sg | 15 / 15 |
| 2CAA | Command Rate | 2N |
| 2CDB | TX (ASRock "VDD_IMC") in mV | 1350 |
| 2CFD | VDD2 (ASRock "VDD_CPU") | Auto (1.45 V) |
| 2D01 / 2D02 | DIMM VDD / VDDQ in mV | 1500 / 1480 |
| 6D | Realtime Memory Timing | Enabled |
| 96 | IGPU Multi-Monitor | Disabled (18 Nov) |
| 8B | Primary graphics adapter | PCI Express |

The parser and the SCEWIN task are published in [raptor-lake-overclock-autopilot](https://github.com/Xipzer/raptor-lake-overclock-autopilot) (`rlh/scewin.py`, `campaign/nvram.py`, `tools/windows/scewin-agent.ps1`).

## The startup failsafe

A SYSTEM task that runs at Windows startup and puts the last good BIOS settings back if a step keeps killing Windows.

How v2 works:

1. Before a risky reboot the controller writes an `armed` marker with the step name, and a list of last good values (token = value).
2. At every startup the task counts starts for that step. One start is normal.
3. A **second start while still armed** means the step's boot never got as far as its test: the step failed.
4. The task then takes a fresh export, writes the last good values into it byte-exact, imports with the password, falls back to a whole saved script only if that fails, logs, writes a `restored` marker, deletes the markers, flushes everything to disk and restarts.

Its limits, all met in practice:

| Situation | Can the failsafe help? | What happened to us |
|---|---|---|
| Crash loop after Windows starts | Yes | tRRD_L 10 (second try): recovered unattended |
| Training fallback (NO TRAIN) | Yes (v2 only) | Several, recovered unattended |
| Windows dies before the task has run twice (about 18 s into each boot) | **No** | tRCD/tRP 48 and tCL 36 needed the owner in the BIOS |
| No POST | **No** | tRDRD_sg 14: five manual power cycles |
| Hard freeze | **No** | E-cores 47x: power button |
| A stale `armed` file left with the task enabled | It does harm | It will restore and restart on the next boot. Disarm after every step and disable the task when the campaign ends |

Rules that came with it:

- **Test the failsafe on purpose with no change** before you rely on it. Our first test found a script error in it.
- Run steps that can cause a no-POST or a freeze only while someone can reach the power button.
- Before every risky reboot, write down the one BIOS field (with its exact number, never "Auto") that undoes the step.
- After any failure: restore, reboot, confirm by export, and only then write the next step. Never write the next step from a failing session.

**Public sources:** we found **no public write-up** of an in-OS BIOS failsafe built on SCEWIN. The pieces (SCEWIN import, startup tasks) are public; the design is ours.

## Manual path, for the person at the keyboard

ASRock: Del > OC Tweaker > DRAM Configuration for timings; OC Tweaker > CPU Configuration for P-core ratio and cache ratio; voltage mode **Fixed** with an explicit voltage.
On this board "Auto" Vcore means adaptive voltage of about 1.38 to 1.46 V, far above a sensible daily value. Never give anyone "set it back to Auto" as a recovery step.
