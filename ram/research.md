# DDR5 research (condensed)

Web research done on 13 November 2026 for this platform: 14900KS, ASRock Z790I Lightning WiFi, 2x24 GB SK hynix 24 Gb M-die at 8400.
Forum posts are self-reported. Where we did our own arithmetic it says "estimate".

Shorthand:

- **OCN-INTEL** = overclock.net, "Official Intel DDR5 OC and 24/7 daily Memory Stability Thread":
  https://www.overclock.net/threads/official-intel-ddr5-oc-and-24-7-daily-memory-stability-thread.1794772/
- **OCN-LIGHT** = overclock.net, "ASRock Z790 NOVA WiFi / Z790I Lightning WiFi Thread":
  https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/
- **ASR-MAN** = ASRock Intel Z790/H770/B760 BIOS Setup Guide:
  https://download.asrock.com/Manual/Software/Intel%20Z790/Software_BIOS%20Setup%20Guide_English.pdf

## 1. ASRock voltage names

| ASRock BIOS name | Rail | Evidence |
|---|---|---|
| VDD_CPU | VDD2 (memory controller I/O, external VR) | ASR-MAN p.50; HWiNFO shows it as VDD2 (OCN-LIGHT page 24, post 475) |
| VDD_IMC | VDDQ_TX (CPU transmitter rail) | OCN-LIGHT page 54, post 1080; same HWiNFO observation |
| VDD_MRC | unclear, leave on Auto | OCN-LIGHT pages 6, 24, 65 |
| VCCSA | System Agent, in the FIVR menu | OCN-LIGHT page 3 |

Verify on your own board: change one value and see which HWiNFO sensor moves.
We did, on 13 November: VDD_IMC set to 1.350 V reads as VDDQ TX 1.350 V, and VDD_CPU set to 1.450 V reads as VDD2_CPU 1.448 V. Both mappings hold on our machine.

On ASRock, VDD_CPU and VDD_IMC follow the XMP DRAM voltage on Auto (OCN-LIGHT post 3789; auto TX 1.40 V, post 3798). So 1.45 V on both is a board rule, not a tuned value.
Above about 1.41 V they need "Voltage Mode: OC Mode" (OCN-LIGHT post 1).

## 2. Daily voltage limits used by experienced tuners

| Rail | What people say for daily use | Sources |
|---|---|---|
| DIMM VDD/VDDQ | Up to about 1.50 to 1.55 V daily on 2x24 M-die; examples run 1.45 to 1.64 V | OCN-INTEL pages 1195, 1617, 1637, 1692; [LTT](https://linustechtips.com/topic/1502804-i-bought-new-2x24gb-ddr5-8000mhz-but-its-m-die-should-i-return/) ("lower voltage tolerance" than A-die) |
| VDDQ_TX | 1.25 to 1.35 V; "1.4V VDDQ_TX is too high for daily" | OCN-LIGHT posts 3783, 3785, 3812; [r/overclocking](https://www.reddit.com/r/overclocking/comments/1bbfbie/ddr5_intel_imc_vdd2_and_ivr_transmitter_vddq_tx/); [ROG forum](https://rog-forum.asus.com/t5/intel-700-600-series/question-regarding-safe-24-7-voltages-for-12th-13th-gen/td-p/957611) |
| VDD2 | 1.40 to 1.45 V; Intel spec typical 1.116 V | [Intel datasheet](https://edc.intel.com/content/www/us/en/design/products/platforms/details/raptor-lake-s/13th-generation-core-processors-datasheet-volume-1-of-2/vdd2-dc-specifications/); [MSI forum](https://forum-en.msi.com/index.php?threads/safe-voltage-range-for-cpu-vddq-and-cpu-vdd2.383716/); OCN-INTEL pages 1013, 1708 |
| VCCSA | 1.22 to 1.36 V in daily examples; "beyond 1.40V degrades it quickly" is a claim with no data | [OCN voltage definitions](https://www.overclock.net/threads/definitions-of-voltages-for-overclocking.1803733/page-3); OCN-INTEL pages 1630, 1708 |

Rail relationships (community heuristics, not specs):

- "need vdd/q delta 80-100 mv vs imc and tx, always" (OCN-INTEL post 32691).
- "Try to keep 100-150mV delta between TX and VDDQ ... Lower SA = Lower TX" (OCN-LIGHT post 2973).

Our XMP state was VDDQ 1.45 = TX 1.45 = VDD2 1.45: zero delta, the opposite of that advice.
That is why our second sweep plan lowers TX and VDD2 first.

## 3. Tuning order from the guides

1. Loose timings, find frequency, then tighten ([integralfx DDR4 guide](https://github.com/integralfx/MemTestHelper/blob/oc-guide/DDR4%20OC%20Guide.md); [Overclockers.com DDR5 guide](https://www.overclockers.com/ddr5-overclocking-guide/)).
2. CPU-side voltages while finding frequency.
3. Secondaries (tRRD, tFAW, tWR) to speed up later testing.
4. tRFC, by binary search. Too tight gives freezes.
5. tWTR through tWRRD on Intel, then tCWL.
6. Tertiaries, one group at a time. "_dg timings have a significant impact on performance, _sg timings doesn't" (Arni90, OCN-INTEL post 1).
7. Primaries: tCL, then tRCD/tRP, then tRAS, then tRC.
8. tREFI last. It is the most temperature-sensitive.

"Lower one timing at a time, test stability, rinse and repeat" (Arni90).

## 4. What others run: 2x24 M-die at 8000 to 8600

| Source | Board | Result | Voltages |
|---|---|---|---|
| OCN-LIGHT post 3812 | Z790I Lightning | 8533 C38 stable, 8600 errors in minutes | SA 1.24, TX 1.28, VDD/VDDQ 1.45 |
| OCN-INTEL post 32322 | Z790I Lightning, 14900KS | 8600 C36, not retrain-validated | VDD 1.62, VDDQ 1.58, SA 1.22, TX 1.32 |
| OCN-LIGHT post 3761 | Lightning, 14900KS SP103, direct die, chiller | 8800 C38; second slot weaker | not given |
| OCN-INTEL post 32731 | Apex Encore, 14900KS SP104, 3 kits | 8400 36-49-49-61, tRFC2 651 (155 ns), tRFCpb 420 (100 ns), turnarounds {15,7,19,19,64,48,12,7} | VDD/VDDQ 1.55/1.50, SA 1.235, TX 1.35, VDD2 1.40 |
| same | same | 8600 36-49-49-61 on 2 of 3 kits | 1.64/1.58, SA 1.355, TX 1.27, VDD2 1.45 |
| OCN-INTEL page 1617 | 14900KF | 8400 C36-49-49-61 daily, 8600 never | 1.50 VDD, 1.44 VDDQ, TX 1.30, SA 1.26 |
| OCN-INTEL page 1725 | | "primaries refuse to go lower than 49-49-49 even with 1.6v+ vdd at 8400" | |

The kit: TechPowerUp's teardown of the Kingston FURY Renegade CUDIMM DDR5-8400 2x24 GB found SK hynix 24 Gb M-die, single-sided, Rambus clock driver, Richtek PMIC
([TechPowerUp](https://www.techpowerup.com/review/kingston-fury-renegade-cudimm-ddr5-8400-48gb-cl40/3.html)).
On Z790 a CUDIMM runs in clock-driver bypass mode ([r/overclocking](https://www.reddit.com/r/overclocking/comments/1iuesjm/z790_cudimm_question/)).

tRFC in ns = clocks x 2000 / MT/s. At 8400: tRFC 680 = 161.9 ns, 630 = 150 ns, 155 ns = about 651 clocks.
kamio1980 runs 155 ns for tRFC2 and 100 ns for tRFCpb on 24 Gb M-die ("never got stuck").

## 5. Intel DDR5 timing rules

| Rule | Source |
|---|---|
| tCL must be even in Gear 2 | Arni90, OCN-INTEL post 1 |
| tCWL = tCL is the safe default. Lower tCWL loads the IMC and shifts tWRRD and so tWTR | integralfx; [KoT guide](https://github.com/KoTbelowall/INTEL-DDR4-RAM-OC-GUIDE-by-KoT/blob/main/guide.md) |
| tRAS >= tRCD + tRTP; tRC = tRP + tRAS | integralfx |
| tFAW = 4 x min(tRRD_S, tRRD_L) is enough | [igor'sLAB](https://www.igorslab.de/en/intel-vs-jedec-ddr5-timings-in-extreme-technical-practice-test/2/) |
| Intel uses tRDPRE, not tRTP | [igor'sLAB](https://www.igorslab.de/en/intel-vs-jedec-ddr5-timings-in-extreme-technical-practice-test/4/) |
| tWR is controlled by tWRPRE / tWRPDEN | Arni90 |
| tWTR_S and tWTR_L are controlled by tWRRD_dg and tWRRD_sg. "can often only tighten 2-6 ticks from stock" | Arni90; integralfx |
| Tertiary minimum on Intel DDR5 is 7. tRDRD_dg and tWRWR_dg typically 8 | Arni90; OCN-LIGHT page 53 |
| tREFI can go to 262143; the only timing where higher is better | Arni90 |

## 6. Temperature

| Claim | Source |
|---|---|
| tREFI is the temperature-sensitive one. 65k when under 45 to 50 C, 32767 good until 70 C | [OCN temperature thread](https://www.overclock.net/threads/ddr5-most-temperature-sensitive-timings.1808660/) |
| 24 Gb M-die at 8600, tREFI 65535: errors from about 47 C | [r/overclocking](https://www.reddit.com/r/overclocking/comments/1odgz3l/investigating_ddr5_trefi_scaling_with_high/) |
| Keep 2x24 kits below 49 C at full load; "don't like going over 42c if i want tight timings" | OCN-LIGHT pages 6 and 54; [OCN M-die thread](https://www.overclock.net/threads/so-as-of-today-hynix-m-die-24-gb-is-the-best-overclocker.1812624/) |
| tRFC was not temperature-dependent in one A-die test; tREFI was | [TechPowerUp](https://www.techpowerup.com/review/ddr5-temperature-variable-analysis/3.html) |
| Gaming heat raises DIMM temperature; test memory with the GPU loaded | integralfx |

## 7. Tests and what they catch

| Test | Catches | Source |
|---|---|---|
| y-cruncher VT3, VST, N63, FFTv4 | IMC, SA/TX/VDD2, heavy combined load; silent I/O errors caught by redundancy checks | [y-cruncher FAQ](https://www.numberworld.org/y-cruncher/faq.html); [version history](https://www.numberworld.org/y-cruncher/version_history.html); [r/overclocking](https://www.reddit.com/r/overclocking/comments/1s5d5v9/why_i_use_ycruncher_first_for_stability_testing/) |
| TM5 (1usmus_v3, anta777) | DRAM-side timing and voltage errors | [TM5 configs](https://github.com/integralfx/MemTestHelper/tree/oc-guide/TM5-Configs) |
| Karhu | DRAM errors over long runs; 93% of errors found within 1 h, 99.3% within 6 h | [ROG forum methodology](https://rog-forum.asus.com/t5/technologies-explained/xmp-amp-expo-memory-stability-methodology-amp-tools/ba-p/1075679) |
| Cold-boot retrains | Training-dependent instability. Profiles failed "after maybe 10-15 cold boots over the course of days and weeks" | OCN-LIGHT post 2974 |

"RAM and IMC stability are quite independent of each other": passing TM5 does not clear the IMC (OCN-INTEL). Run both kinds.

## 8. Platform caveats

- **SA bug**: some 14th-gen chips hard-freeze above a chip-specific VCCSA (about 1.12 to 1.30 V). See [quirks.md](quirks.md).
  Sources: [OCN SA freeze thread](https://www.overclock.net/threads/14900k-14900k-cpu-sa-voltage-issue-freezing-problem-and-solved.1808469/); OCN-INTEL pages 1287, 1307, 1319, 1482.
- **Vmin shift**: Intel traced Raptor Lake degradation to a clock tree circuit in the IA core, aged by elevated voltage and temperature
  ([Intel](https://community.intel.com/t/intel-core-13th-and-14th-gen-desktop-instability-root-cause-update/127016);
  [Puget Systems](https://www.pugetsystems.com/labs/articles/impact-of-intel-microcode-0x12b-on-content-creation-performance/)).
  No Intel statement covers SA, VDD2 or TX limits.
- **PMIC lock**: some Kingston FURY kits are capped at 1.435 V when the PMIC is JEDEC-locked (ASR-MAN; OCN-LIGHT post 475). Ours runs XMP at 1.45 V, so it is not locked.
- **On-die ECC**: DDR5 corrects single-bit errors inside the chip silently
  ([ATP](https://www.atpinc.com/tw/blog/ddr5-what-is-on-die-ecc-how-is-it-different-to-traditional-ecc)). They never reach WHEA.
- **WHEA event 19** means the CPU-side configuration is unstable ([overclock.net](https://www.overclock.net/threads/about-whea-event-19-warnings.1536052/)).
- **Is 8600 worth it?** About +2.4% bandwidth, for more SA, more DIMM voltage and more risk. On this board 8533 to 8600 is reported for good chips; several owners stop at 8400 (OCN-LIGHT posts 3743, 3789, 3809, 3812). A binning of 28 14900KS chips found big IMC spread ([overclock.net](https://www.overclock.net/threads/results-of-binning-28-x-14900ks-cpus.1811913/page-2)).

## 9. Second round: floors and rules for the late steps

Done on 13 November before the heat-limited and risky steps. One clock at 8400 is 0.2381 ns.
What this research predicted for each step, and what happened, is in [predictions.md](predictions.md).

Main sources:
[Pakhtunov's DDR5 guide, rev. 5.04](https://pakt.cc/en/ddr5/) ([timings](https://pakt.cc/en/ddr5/timings/), [formulas](https://pakt.cc/en/ddr5/formulas/), [PDF](https://pakt.cc/files/DDR5_5.04_2026_EN.pdf));
the [SK hynix 32 Gb M-die datasheet](https://uttc.com.tw/wp-content/uploads/2026/01/DDR5_SDRAM_32Gb_M-die_Rev.1.2-1_H5CG58MHBDX051N.pdf) (no public 24 Gb datasheet found);
[igor'sLAB](https://www.igorslab.de/en/intel-vs-jedec-ddr5-timings-in-extreme-technical-practice-test/);
[Buildzoid on refresh overhead](https://www.patreon.com/buildzoid/posts/low-effort-rank-77403831); OCN-INTEL post 1; integralfx.

| Topic | What the sources say | What we found |
|---|---|---|
| tRAS floor | Precharge waits for both tRAS and tRCD + tRTP. Below tRCD + tRTP, lower tRAS gains nothing. Hidden tRC = tRAS + tRP (Pakhtunov; integralfx) | Floor here is 72 to 73 with the read-back tRTP of 23. We first used a wrong tRTP |
| tRTP | DRAM minimum 12 clocks. Intel's controller uses tRDPRE, range 4 to 32. Pakhtunov runs 14 (datasheet; igor'sLAB) | Untuned at 23. An explicit value reads back one lower than entered |
| tWR | The BIOS field drives the controller's write-to-precharge wait (tWRPRE, tWRPDEN). The DRAM's own write recovery is 30 ns and does not shrink. Pakhtunov keeps 48 (OCN-INTEL post 1; datasheet) | 20 passes. It does not move tRTP on this board |
| tCWL | At the DRAM, CWL = CL - 2. Lowering tCWL with tWRRD fixed raises tWTR (datasheet; Pakhtunov; integralfx) | tCWL 38 at tCL 40 and tCWL 36 at tCL 38 both pass. tWTR rose by the same amount, as stated |
| tWTR | tWTR = tWRRD - tCWL - 10 (Pakhtunov) | On this board the constant is 6 |
| tRRD_L | Datasheet at 8000 to 8400: max(8 clocks, 4.0 ns) = 17 clocks. The 5 ns figure is the older DDR5-4800 value. Starting point tRRD_S + 2 to 4. Gains "in the per mille range" (datasheet; igor'sLAB; Pakhtunov) | 11 passes. Already far below the datasheet |
| tCCD_L and tRDRD_sg | max(8 clocks, 5 ns) = 21 clocks at 8400. "Start with the trained value" for tRDRD_sg (datasheet; Pakhtunov) | Trained is 16. 15 passed, 14 gave no POST |
| tRFC, 24 Gb | JEDEC tRFC2 220 ns, tRFCsb 190 ns. Pakhtunov's working range at 8200+ is 688 to 720 clocks. Community floor about 160 ns | 660 (157 ns) passes, 630 (150 ns) does not train |
| tRFCpb | About tRFC - 30 x MT/s / 2000 (Pakhtunov, who leaves it on Auto). Arni90: ratio about 0.69. It was unclear whether this BIOS uses it | 420 (100 ns, ratio 0.64) passes, with a latency gain |
| Refresh overhead | Roughly tRFC / tREFI of the time goes to refresh. Returns shrink near 1% (Buildzoid) | See the table below |
| tREFI and heat | Above about 30000 use directed airflow. One 2x24 GB kit errored at about 47 C at 131071 (Pakhtunov) | 65535 passes at 52 / 57.5 C in short runs |
| tCL | Below CL36 above about 7600 needs "a very large VDD increase". CL responds to DIMM VDD, not to TX or VDD2 (Pakhtunov) | 38 fails at 1.45 V, passes at 1.50 / 1.48 V |
| Ring and E-cores | Ring is part of the latency path. P-cores, E-cores and ring share one voltage rail. Pakhtunov runs a 50x ring ceiling on a 14900KS, lower with HT on ([CPU guide](https://pakt.cc/files/CPU_2.01_2026_EN_FINAL.pdf); [SkatterBencher](https://skatterbencher.com/2023/12/16/skatterbencher-67-intel-core-i9-14900k-overclocked-to-6200-mhz/); [Chips and Cheese](https://chipsandcheese.com/p/alder-lake-e-cores-ring-clock-and-hybrid-teething-troubles)) | Ring 51 fails. E 46 passes, E 47 freezes |
| What to suspect | No POST or training loops: VDD2, TX, SA first. Early write errors: VDDQ, TX. Errors after warm-up: VDD, tREFI, tRFC, cooling (Pakhtunov) | Our warm-up errors were tCL 38 at too little VDD |

Refresh overhead at our values (estimate, tRFC / tREFI):

| tRFC | tREFI | Time spent refreshing |
|---|---|---|
| 922 | 32767 | 2.81% (XMP) |
| 680 | 32767 | 2.08% |
| 680 | 49152 | 1.38% |
| 660 | 65535 | 1.01% (kept set) |

Method advice from the same guide that we adopted or still owe:

- Record the values the board applied after boot, not the ones you typed. "Auto is not an empty value."
- Measure the gain before paying for it. Use the median of several runs. A difference smaller than run-to-run spread is not a gain.
- Split memory errors from CPU errors with an in-cache y-cruncher run against a large one.
- Karhu finds about 75% of errors within 10 minutes and 98% within 3 hours. Our 7-minute runs are a screen, not a proof.
- Test at the DIMMs' daily temperature plateau, and repeat after a cold boot, a reboot and sleep.

## 10. Gaps

- Intel's VCCSA maximum for Raptor Lake: not found.
- What VDD_MRC does on this board: unknown.
- Memory-OC changes in BIOS 10.01: none reported.
- How VBS affects tester coverage: no authoritative source.
