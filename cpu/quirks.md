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

## Errors and crashes without a WHEA record

Three of our failures were crashes with no corrected error logged (59x on P0/P1, 59x ring 45x, 57x at 1.330 V for 30 min).
A core fault that the CPU cannot correct can take the process down before anything is logged. Treat a silent crash as a fail.

## Cinebench resets its own affinity

`start /affinity` does not hold cores out of Cinebench 2026: it resets its affinity and uses every core.
Holding cores out needs a job object with an affinity limit (Shakedown does this), or re-applying affinity in a loop.

## Tj Max 115 C

This board lets you set Tj Max and the fast throttle threshold up to 115 C, and ours were set that way.
Several validation runs had P7 above 100 C. Intel's default is 100 C. Decide this for yourself; we note it because it changes what "no throttling" means.
