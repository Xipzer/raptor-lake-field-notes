# Hardware

## Projector

### Platform

| Part | Detail |
|---|---|
| CPU | Intel i9-14900KS, delidded, direct-die. BIOS SP rating: CPU 96, P-core 101, E-core 86 |
| Fused data | All 8 P-cores show ratio 62 and VID 1.509 V in the BIOS CPU indicator. Fused favoured cores: P4 and P5 (max ratio 62). The rest: 59 |
| Board | ASRock Z790I Lightning WiFi (mini-ITX, 2 DIMM slots, single 8-pin EPS) |
| BIOS | 10.01, microcode 0x12F |
| Vcore regulator | Renesas RAA229131 (source: [Tom's Hardware review](https://www.tomshardware.com/pc-components/motherboards/asrock-z790i-lightning-wi-fi-review)) |
| Super I/O | Nuvoton NCT6796D (fans, board temperatures) |
| RAM | Kingston FURY Renegade 2x24 GB DDR5-8400 CL40-52-52-132, part KF584CU40RSA-24. Single rank, SK hynix 24 Gb M-die, Richtek PMIC. It is a CUDIMM, which runs in clock-driver bypass on Z790 |
| GPU | RTX 5090 Founders Edition with a full-cover water block |
| PSU | 1300 W Platinum |
| OS | Windows 11 with Hyper-V and VBS on (WSL2 runs all the time) |

### Cooling

- CPU block: SuperCool direct-die, newer hybrid frame (no through-board mounting). Nickel cold plate, never polished.
- Liquid metal on the bare die. The old application was still liquid and shiny after months.
- Radiators: 3 x 360 mm, 30 mm thick, 16 FPI (top, bottom, side).
- Fans: 9 x Arctic BioniX P12 A-RGB (2026 model, 3,000 rpm max), three chains of three on motherboard headers.
- Pump: D5 pump/reservoir combo with an LCD that shows coolant temperature.
- Case: Lian Li O11 Dynamic, run with all panels and filters off.
- RAM: no heatsink fan at first. A 120 mm fan was aimed at the DIMMs from the night of 12 to 13 November. A RAM water block is planned.

### Loop order

```
pump -> bottom radiator -> GPU -> top radiator -> CPU -> side radiator -> pump
```

The coolant sensor sits at the pump. That is the coolest point in the loop.
The CPU block sees water that has already taken the GPU's heat and passed one radiator.
So with the GPU loaded, the CPU's inlet water is warmer than the reading. See [cpu/thermals.md](cpu/thermals.md).

### Fan curve

Coolant-driven, run by a small tray app through the Super I/O:

- 40% at or below 30 C, linear up to 80% at 45 C.
- 2 C hysteresis, about 30 s of averaging.
- Capped at 80% because two fan chains share 1 A chassis headers. A BioniX chain draws about 0.9 to 1.3 A at 100% on a radiator. A SATA-powered fan hub will lift the cap.

### Sensors and where each reading comes from

| Reading | Source |
|---|---|
| Per-core temperature, real clock (APERF/MPERF), package power | CPU MSRs, read through PawnIO's signed IntelMSR module |
| Corrected CPU errors per core | Windows WHEA-Logger event 19 (APIC ID mapped to core) |
| Coolant | the pump's own sensor, read over USB HID |
| GPU power and temperature | NVIDIA NVML (read only) |
| DIMM temperature | SPD hub thermal sensor on each DIMM, read over SMBus (PawnIO SmbusI801 module). Cross-checked with HWiNFO |
| Vcore, VccSA, VDD2, TX, DIMM VDD | HWiNFO |
| Effective BIOS settings | SCEWIN export of the BIOS setup questions |

Core mapping on this CPU: P0..P7 = APIC 0, 8, 16 .. 56. E0..E15 = APIC 64, 66 .. 94.

### BIOS baseline before tuning (from photos of the BIOS)

| Setting | Value |
|---|---|
| P-core ratio | All Core 59 (options: Auto / All Core / Per Core / Specific Per Core) |
| E-core ratio | All Core 45 |
| Cache (ring) | 50, Ring to Core Ratio Offset Disabled |
| CPU Core/Cache Voltage | Fixed Mode 1.360 V (external regulator) |
| Load-line calibration | Level 1 (flattest of Auto/1/2/3/3.5/4/5) |
| IA AC/DC load line | 0.50 / 0.50 |
| System Agent | Override 1.250 V |
| VDD_CPU (VDD2) / VDD_IMC (TX) | 1.450 / 1.450 (Auto, follows XMP) |
| Power limits | PL1/PL2 4095.875 W, Core Unlimited Current Limit Enabled |
| Tj Max / Fast Throttle | 115 C / 115 C |
| C-states | Core C-states on (C6/C7 Auto), package C-state off |
| SpeedStep, Speed Shift, Turbo Boost Max 3.0 | Enabled |
| TVB Ratio Clipping | Disabled (option exists). TVB Voltage Optimizations Enabled. eTVB Disabled |
| Undervolt Protection | Enabled |
| Boot Failure Guard | Enabled, count 4 |
| Memory | XMP Profile 1, DDR5-8400 40-52-52-132 at 1.45 V, Gear 2, timings Auto. VDD/VDDQ 1.455 V |

A note on Tj Max: Intel's default is 100 C. This board allows 115 C, and it was set that way before we started.
Some cores reached 106 C in validation. That is a choice, not a recommendation.

Idle package power with package C-states off and fixed voltage: 80 to 140 W.

## Printer

| Part | Detail |
|---|---|
| CPU | i9-14900KS, SP 110, direct-die, liquid metal |
| Board | ASRock Z790I Lightning WiFi, microcode 0x12F |
| Settings | 59x all P-cores, 46x E-cores, ring 50x, BCLK 99.8 MHz, fixed Vcore about 1.35 V (1.344 V measured at about 400 W), very flat load line, power unlimited |
| C-states | On (idle package 35 to 59 W) |
| RAM | DDR5-6600, VccSA about 1.0 V, TX 1.40 V, VDD2 1.40 V |
| Record | 0 corrected CPU errors in 120 days under a 200 to 300 W 24/7 server load |

The catch: every Printer Cinebench log was 3 to 5 minutes on 24 to 28 C water.
Printer has never been tested at Projector's heat-soaked water temperature. "Printer is stable at 59x" is proven only in those conditions.
