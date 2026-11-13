# CPU thermals

Direct-die with liquid metal, 14900KS at about 400 to 420 W package power in Cinebench 2026.

## Core-to-water per watt

We track the gap between core temperature and water temperature, divided by package power.
It is the best single number for "is the cold plate contact still good?", because it removes both water temperature and load.

- **12.0 to 12.6 C per 100 W**, flat across 7 hours of about 400 W load (12 November, 03:48 to 10:57).
- Flat means no liquid-metal degradation over that period, even with cores at 90 to 106 C for hours.

We had a scare that evening: the gap between the hottest core and the water looked 5 to 7 C worse than in the morning.
The per-watt figure was still flat. The difference was power and inlet temperature, not the interface.
Lesson: compare per watt, at the same GPU load, before you repaste.

The hottest-core gap is larger than the per-watt average suggests. Two examples from the ledger:

| Run | Hottest core | Water max | Gap | Package |
|---|---|---|---|---|
| 57x 1.340 V, 30 min, run 1 | P7 103 C | 39.2 C | about 64 C | 413 W |
| 57x 1.340 V, 30 min, run 3 | P7 106 C | 39.5 C | about 66 C | 418 W |

## Fixed per-core offsets

The same offsets showed up in every run, relative to the core average:

| Core | Offset |
|---|---|
| P7 | about +6 C (always the hottest) |
| P3 | about +3 C |
| P5 | about +2 C |
| P0, P1, P6 | about -3 to -4 C |

Spread across P-cores: about 10 C. Steady all night. A spread that suddenly changes is a better sign of a contact problem than a hot run.

P7 being the hottest made it fail first at 59x under heat, even though it is not one of the weaker cores when cool.

## GPU heat

- The CPU block sits after the GPU and the top radiator. With the GPU at about 530 W, the CPU cores read **about 3 to 4 C hotter at the same pump reading**.
- Starting from about 19 C water, GPU heat warms the loop by about 2.3 C per minute until the plateau.
- A CPU test with the GPU idle is a colder test than daily use on a shared loop.

## Loop plateau

| Load | Fans | Water plateau |
|---|---|---|
| about 930 W (CPU about 400 W + GPU about 530 W) | 60 to 65% | about 36.5 C early in the night |
| about 950 W after 3.5 hours (room warming) | 65% | 39.5 C |
| about 600 W, GPU only | 45% fixed | 32 to 36 C |

Rough model from these numbers (estimate): about 1 C of water per 60 to 70 W at mid fan speeds, for this 3 x 360 mm loop with open panels.

The fans only reached 65% during validation because the curve was capped at 80% at 45 C. A steeper curve would lower the plateau.

## Heat and stability, in numbers

- At 58x, P2 failed with its core at 84 C at 1.340 V and at 89 C at 1.360 V. So +20 mV bought about 5 C.
- Public rough figure for Raptor Lake (SkatterBencher, 13900K): about 1 mV per C, and about 29 mV per ratio bin.
  That makes one bin worth roughly 29 C of margin at fixed voltage
  ([source](https://skatterbencher.com/2022/12/04/skatterbencher-49-intel-core-i9-13900k-overclocked-to-6100mhz/)).
  Our P2 numbers (about 4 mV per C) are steeper than that. Your chip will differ again.
- Cooling helps, but on its own it will not buy a full ratio bin.
