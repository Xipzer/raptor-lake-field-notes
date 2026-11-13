# CPU quirks

## ASRock "Fixed" Vcore lives at the VRM, not in the CPU

The BIOS item "CPU Core/Cache Voltage: Fixed Mode" is described as input voltage "by the external voltage regulator".
We confirmed this on the machine:

- Reading the CPU's overclocking mailbox (MSR 0x150) showed the cores in **adaptive mode with target 0**, the ring adaptive,
  and only the System Agent domain fixed at 1,250 mV (the SA override).
- So the 1.340 V the cores receive is set at the board's regulator. The CPU does not know about it.

Consequences:

- Software that writes voltage offsets through the mailbox (XTU-style, IntelVoltageControl, our own experiment) does nothing useful in this mode.
- TVB Voltage Optimizations and V/F point offsets change the CPU's voltage request only. With Fixed Mode they have no effect.
- Runtime undervolting also runs into Intel's Undervolt Protection and the hypervisor ([Intel 000094219](https://www.intel.com/content/www/us/en/support/articles/000094219/processors.html)).
  BIOS changes are always allowed.

## A-Tuning changes ratio and Vcore live

ASRock's own Windows utility (A-Tuning) changes the all-core P-core ratio and the Fixed Vcore **live**, with Hyper-V and VBS running.
Our research had said no runtime path existed on this board. The owner found it by opening the program.

- It only showed an all-core ratio, not per-core.
- "Auto apply when program starts" was off, so a reboot returns the BIOS profile. That is a free safety net while testing.
- It has no command line and no UI automation interface. We drove it with an elevated helper that posts clicks to its `-` / `+` and Apply buttons
  (one press = 5 mV) and captures the window to confirm the value. Check the capture every time: a click that did not land looks exactly like a test that passed.

## Per-core ratios on this board

- "CPU P-Core Ratio" options: Auto, All Core, Per Core (by number of active cores), Specific Per Core (a ceiling per physical core).
- Specific Per Core worked as entered: measured clocks matched (5,870 MHz at 59x, 5,770 MHz at 58x).
- The BIOS shows each core's fused max ratio. Ours: cores 4 and 5 at 62 (favoured), the rest 59.
- Speed Shift and Turbo Boost Max 3.0 must be on for Windows to prefer the faster cores.

## Load-dependent ratio ideas, and why we dropped them

The owner first wanted 59x for light work and a lower ratio only under saturated, hot load. We looked at every mechanism:

| Mechanism | Verdict here |
|---|---|
| AVX2 Ratio Offset | Does not trigger in Cinebench R23 or 2024 (sources: SkatterBencher). Covers Prime95 and maybe OCCT AVX2 only |
| Active-core turbo ratios ("Per Core") | Useless with fixed load: a game keeps almost all cores awake, so it looks the same as Cinebench |
| TVB Ratio Clipping | Intel's threshold (about 70 C), no user threshold on ASRock. Reported not to work with Specific Per Core |
| Lower Tj Max | A user-set hot threshold, but it is throttling by another name. Smoothness unverified |
| Current limit | Works on predicted current. Acts at idle too with C-states off |
| Power limits | See below |

**Why power limits fail with fixed voltage.** With Vcore fixed, power barely changes with ratio: about 5 W per ratio bin at this load.
To pull the clock down by one or two bins under Cinebench, the limit has to sit just under the Cinebench draw.
But the CPU's response to hitting a power limit is to cut the ratio until power fits, and at fixed voltage each bin saves so little
that a limit lands the clock about 12 bins low instead of 1 or 2. That is a much bigger loss than simply running 57x.

The owner then set a rule: **one universal overclock, stable in every workload, no dynamic pull-back.**
His reason: AVX2 is in games and everyday software now, so a clock that drops under AVX2 is a clock you never really run.

A possible future tool, not built: a software ratio switcher (59x light, 57x heavy) driven by temperature or package power through the mailbox ratio field.
It would first need proof that 59x is stable at medium load, which we do not have.

## Ring 45x did not help

Dropping the ring from 50x to 45x at 59x / 1.380 V did not delay P2's failure (3:34 vs 3:37).
On this chip the P-cores were the limit, not the ring. The ring stays at 50x. Your chip may differ: test ring 45x first if your errors are not tied to specific cores.

## Ring 51x does not hold, and the memory test cannot see it

- Ring 51x at 57x / 1.340 V: Cinebench with GPU heat crashed at 2:24. At 1.350 V: crashed at 4:13. No WHEA record either time.
- The memory stress test (y-cruncher FFTv4, N63, VT3) passed twice at each voltage.
- So 50x is this chip's ring ceiling with Hyper-Threading and E-cores on, at a voltage and temperature we accept.
  That matches what one detailed public guide runs on its own 14900KS: a 50x ring ceiling, with the note that Hyper-Threading needs a lower ring
  ([Pakhtunov, CPU guide v2.01](https://pakt.cc/files/CPU_2.01_2026_EN_FINAL.pdf)).
- Ring, E-cores and P-cores share one voltage rail on this CPU ([SkatterBencher #67](https://skatterbencher.com/2023/12/16/skatterbencher-67-intel-core-i9-14900k-overclocked-to-6200-mhz/)).
  With a fixed voltage, a faster ring gets no extra voltage. It only loses margin.

## E-cores: 46x passes, 47x freezes

- 46x passed two memory runs and 10 minutes of Cinebench with GPU heat (10,059.55).
- 47x hard-froze the PC about 3 minutes into the memory test. The clock on the lock screen stopped. Nothing was logged except the unclean shutdown.
- A freeze is worse than a test error for an unattended sweep: nothing restarts the machine. See [../ram/method.md](../ram/method.md).
- The E-core ratio has no live read-out in this BIOS. We confirmed 46x from the measured clock (4,580 MHz).

## Load-line calibration and AC/DC load line (research, not yet tested here)

Our profile is Fixed 1.340 V, LLC Level 1, AC and DC load line both 0.50 mOhm. We asked whether that combination makes sense.
This is web research from 13 November. Nothing in this section was measured on our machine yet.

**What the ASRock LLC levels are.** ASRock publishes no numbers. Level 1 is the flattest.
Community measurements on the sister board (Z790 Nova) from regulator telemetry:

| ASRock level | Load line (measured by owners) |
|---|---|
| Level 1 | about 0.20 mOhm |
| Level 2 | about 0.60 mOhm |
| Level 3 | about 0.74 to 0.88 mOhm |
| Level 4 | about 1.10 mOhm (Intel's spec value) |
| Level 5 | about 1.70 mOhm |

Sources: [Nova/Lightning owners' thread, page 25](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-25)
and [page 62](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-62);
[r/overclocking](https://www.reddit.com/r/overclocking/comments/1bsh3hs/acdc_ll_tuning_for_dummies/);
level naming from [Elmor](https://www.elmorlabs.com/2019-09-05/vrm-load-line-visualized/)
and the [ASRock manual](https://www.manualowl.com/m/ASRock/Z790-PG-ITX%2FTB4/Manual/669857?page=53).
Thin evidence: two or three owners, on the Nova, not on the Lightning. The Lightning has a different regulator and phase count.

**In fixed mode, AC and DC load line do not change the delivered voltage.**
Delivered voltage is the set voltage minus LLC resistance times current. AC LL and DC LL are not in that sum.
They only change the VID the CPU reports, and through it the reported package power.
Sources: [r/intel](https://www.reddit.com/r/intel/comments/bz7vdo/max_152v_vid_auto_voltages_ac_and_dc_loadlines/);
[overclock.net](https://www.overclock.net/threads/gigabyte-z390-aorus-owners-thread.1711478/page-453);
an ASRock owner in fixed mode: "Modifications to AC/IA DC are having no impact" (owners' thread, page 25).

- So our 0.50 / 0.50 is neutral for the voltage the cores get.
- Reported package power may be off by tens of watts (estimate: each 0.1 mOhm of DC LL moves reported VID about 31 mV at about 315 A).
  To fix the reporting, tune DC LL until reported VID matches the regulator's VR VOUT under load
  ([source](https://www.overclock.net/threads/msi-lite-load-and-mapping-to-cpu-ac-dc-load-lines.1805086/page-2)). This is cosmetic if power limits are off.
- Possible side effect (estimate, not confirmed): the CPU still computes a VID in fixed mode, and a high AC LL could trip VID-based limits and stretch clocks.
  Keeping AC LL low is the safe side. A falling Cinebench score with no crash would be the sign.

**The board's Vcore sensor reads at the socket, so it reads high under load.**
Owners of this board family say so directly ("VCore is demonstrably socket sense",
[owners' thread, page 27](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-27)).
A flat load line then looks like it "overshoots" under load. It does not.
We read 1.336 V at 1.340 V set and 1.344 V at 1.350 V set. Estimate: the die sees roughly 1.27 to 1.29 V at about 410 W.
Read HWiNFO's **VR VOUT** (from the regulator) for any load-line comparison, not Vcore.

**For a fixed voltage, Level 1 is the least-bad level.** Estimate, from the numbers above:

| Level | Set voltage needed for the same loaded die voltage | Voltage at idle |
|---|---|---|
| Level 1 (0.20 mOhm) | about 1.34 V (ours) | about 1.33 to 1.34 V |
| Level 2 (0.60 mOhm) | about 1.47 V | about 1.46 V |
| Level 3 | about 1.51 to 1.56 V | 1.5 V or more |

A droopier level in fixed mode means a much higher voltage at idle and light load.
Intel names elevated voltage at idle and light load as a cause of Raptor Lake degradation
([Intel root cause](https://community.intel.com/t/intel-core-13th-and-14th-gen-desktop-instability-root-cause-update/127016)).
The cost of a flat load line is more overshoot when the load is released. That overshoot lasts microseconds at near-zero current
([Elmor](https://www.elmorlabs.com/2019-09-05/vrm-load-line-visualized/); [Buildzoid](https://www.youtube.com/watch?v=eUzbNNhECp4)).
Nobody has published scope captures for this board, so its size here is unknown.

**The bigger risks in our profile are not the load line.**

1. **Fixed mode bypasses the idle-voltage microcode fix.** Microcode 0x12B and 0x12F lower voltage requests at idle and light load
   ([Intel 0x12F note](https://community.intel.com/t/intel-core-13th-and-14th-gen-vmin-shift-instabilty-update-new-microcode-update-0x12f/132113)).
   With a fixed voltage the regulator holds about 1.33 V all day, whatever the microcode asks for.
2. **103 to 106 C under load.** Intel's statement names elevated voltage and temperature together.
3. **About 410 W through one 8-pin EPS connector.** This board has only one
   ([Tom's Hardware](https://www.tomshardware.com/pc-components/motherboards/asrock-z790i-lightning-wi-fi-review);
   [PC Gamer](https://www.pcgamer.com/hardware/motherboards/asrock-z790i-lightning-wifi-motherboard-review/)). Check the plug is fully seated.

**The safer daily alternative, not yet tested here:** adaptive voltage with LLC Level 3 or 4, a DC load line calibrated to VR VOUT,
a low AC load line and a negative offset until the loaded voltage matches the fixed profile. That lets the CPU drop its voltage at idle
(an owner of the sister board reports idle voltage from about 0.69 V in that mode; owners' thread, page 162).
Undervolt Protection may block the offset until it is turned off, so check that the offset really took effect.

One more ASRock report worth checking on every edit: "there is a bios input bug where LLC automatically sets to level 1 after changing the voltage"
([owners' thread, page 162](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-162)).

## A monitoring tool can take the fans away from the fan controller

While HWiNFO was open, our fan-control tray app lost the Super I/O chip both programs read, and handed the fans to the BIOS curve.
The motherboard fans ran at about 3,000 rpm for the 45 s or so. Any fan controller that reads the same chip can do this.
It also means a voltage capture with HWiNFO changes the cooling for as long as it runs. Ours overlapped a Cinebench run that then crashed; we do not know that it mattered.

## Errors and crashes without a WHEA record

Three of our failures were crashes with no corrected error logged (59x on P0/P1, 59x ring 45x, 57x at 1.330 V for 30 min).
A core fault that the CPU cannot correct can take the process down before anything is logged. Treat a silent crash as a fail.

## Cinebench resets its own affinity

`start /affinity` does not hold cores out of Cinebench 2026: it resets its affinity and uses every core.
Holding cores out needs a job object with an affinity limit (Shakedown does this), or re-applying affinity in a loop.

## Tj Max 115 C

This board lets you set Tj Max and the fast throttle threshold up to 115 C, and ours were set that way.
Several validation runs had P7 above 100 C. Intel's default is 100 C. Decide this for yourself; we note it because it changes what "no throttling" means.
