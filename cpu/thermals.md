# CPU thermals

Direct-die with liquid metal, 14900KS at about 400 to 420 W package power in Cinebench 2026.

## Core-to-water per watt

We track the gap between core temperature and water temperature, divided by package power.
It is the best single number for "is the cold plate contact still good?", because it removes both water temperature and load.

- **12.0 to 12.6 C per 100 W**, flat across 7 hours of about 400 W load (12 November, 03:48 to 10:57).
- Flat means no liquid-metal degradation over that period, even with cores at 90 to 106 C for hours.
- Flat is not the same as good. When the block was opened on 14 November the contact turned out to have been poor from the start of this application. See "The repaste" below.

We had a scare that evening: the gap between the hottest core and the water looked 5 to 7 C worse than in the morning.
The per-watt figure was still flat. The difference was power and inlet temperature, not the interface.
Lesson: compare per watt, at the same GPU load, before you repaste.

The hottest-core gap is larger than the per-watt average suggests. Two examples from the ledger:

| Run | Hottest core | Water max | Gap | Package |
|---|---|---|---|---|
| 57x 1.340 V, 30 min, run 1 | P7 103 C | 39.2 C | about 64 C | 413 W |
| 57x 1.340 V, 30 min, run 3 | P7 106 C | 39.5 C | about 66 C | 418 W |
| 57x 1.340 V, E 46x, 10 min (13 Nov) | P7 105 C | 37.1 C | about 68 C | 417 W |
| 57x 1.350 V, ring 51x, crashed at 4:13 (13 Nov) | P7 106 C | 36.4 C | about 70 C | 429 W |

The last row is why we did not chase ring 51x with more voltage: +10 mV added about 12 W and put P7 at 106 C on water 3 C cooler than the validation runs.

## Fixed per-core offsets

The same offsets showed up in every run, relative to the core average:

| Core | Offset |
|---|---|
| P7 | about +6 C (always the hottest) |
| P3 | about +3 C |
| P5 | about +2 C |
| P0, P1, P6 | about -3 to -4 C |

Spread across P-cores: about 10 C on 12 November. Steady all night. A spread that suddenly changes is a better sign of a contact problem than a hot run.
By 14 November it was 19 C (P1 87 C to P7 106 C at 421 W), and the E-cores were hotter than P1. That is what sent us to open the block.

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

## The repaste (14 November)

The owner opened the block and redid the liquid metal with a fresh syringe.

What was found (from the owner's photos):

- A matt, granular crust over the centre and upper part of the die, the P-core area. Shiny wet metal only at the bottom third.
- The old application was only 3 to 4 days old, from a rebuild earlier that week. At that job the metal would not wet part of the die despite a generous amount, and old metal may have been drawn back into the syringe.
- Our reading: most likely oxidised metal reused from the syringe. Oxide-laden metal is grainy and does not wet. Conductonaut Extreme also wets poorly and tends to bead. Not slow drying.

Same load before and after: y-cruncher on all 32 threads for 100 seconds (Measured):

| | Before | After |
|---|---|---|
| Water | about 26 to 28 C | 24.0 to 25.6 C |
| Hottest core | 95 C at 430 W (P6) | 81 to 83 C at 440 to 513 W (P3, P5, P7) |
| Hottest core above water, per 100 W | about 16 C | about 13 C |
| P-core spread | 19 C (heat-soaked, Cinebench) | 11 C (P6 72 to P3 83 C) |

The hot spot moved from P6 / P7 to P3 / P5 / P7. The "after" runs are short, not heat-soaked, so this is a like-for-like short-load comparison only. A re-check in about a week is planned.

### What liquid metal actually does over time

- **It does not dry. It alloys.** Gallium reacts with the metal it touches. On a nickel-plated surface a thin nickel-gallium layer forms and then stops growing.
  A two-year test by der8auer found the contact still working after the reaction ([TechPowerUp](https://www.techpowerup.com/353807/long-term-liquid-metal-tests-reveal-promising-longevity-despite-reactions)).
- So "dried out" is usually one of: gallium taken up by a fresh surface, oxidised metal, or metal that never wetted in the first place.
- The die's backside metal can stain brown. Harmless. **Do not polish the die.** A grey patch on a nickel lid is the alloy layer and is permanent.
- Judge a repaste per watt above water, at the same load, never by one hot run.

### Printer's repaste, for comparison (Measured on Printer, Cinebench, rows above 250 W)

| Log | Hottest core above water, per 100 W | Hottest P-core vs the rest |
|---|---|---|
| After the previous repaste, 10 Oct | 12.3 C | level |
| Before cleaning, 12 Nov | 13.9 C | P7 about 8 C hotter |
| Before cleaning, 14 Nov | 14.0 C | P7 about 8 C hotter |
| After the repaste, 15 Nov | 12.7 C | level |

Printer's die was crusted all over after about 5 weeks. The repaste took its hottest core from 14.0 to 12.7 C per 100 W above water and cleared the P7 hot spot.

## GPU, for reference

The RTX 5090 die runs 18 to 20 C above the coolant at 600 to 700 W (owner's reading). Thermal Grizzly publishes 18.8 K average and 19.4 K maximum for Conductonaut Extreme on its own RTX 5090 block (Source: the vendor's own published figures; we did not keep the link).
So the GPU contact is at the published level and was left alone.
