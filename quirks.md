# Quirks index

Every quirk we hit while tuning an i9-14900KS on an ASRock Z790I Lightning WiFi with 2x24 GB DDR5-8400 (SK hynix 24 Gb M-die), with its remedy, where the details are, and whether anyone else has reported it.
Most of it applies to any 13th or 14th gen Intel desktop on Z790 with DDR5. Board-specific items say so.

"Public" column: a link means a public source reports the same thing. **None found** means we searched and found nothing; treat it as one machine's finding.

## CPU

| Quirk | Symptom | Remedy | Public | Details |
|---|---|---|---|---|
| ASRock "Fixed" Vcore is set at the VRM | Mailbox shows cores adaptive with target 0; XTU-style offsets do nothing | Change Vcore in the BIOS or A-Tuning, not through the mailbox | [Intel UVP note](https://www.intel.com/content/www/us/en/support/articles/000094219/processors.html) (related) | [cpu/quirks.md](cpu/quirks.md#asrock-fixed-vcore-lives-at-the-vrm-not-in-the-cpu) |
| A-Tuning changes ratio and Vcore live | A runtime path exists despite research saying no | Use it for sweeps; leave "Auto apply" off so a reboot reverts | None found | [cpu/quirks.md](cpu/quirks.md#a-tuning-changes-ratio-and-vcore-live) |
| Fixed Vcore bypasses the 0x12B / 0x12F idle fix | About 1.33 V at idle all day | Accept it knowingly, or use adaptive with a calibrated load line | [Intel 0x12F](https://community.intel.com/t/intel-core-13th-and-14th-gen-vmin-shift-instabilty-update-new-microcode-update-0x12f/132113) | [cpu/quirks.md](cpu/quirks.md#load-line-calibration-and-acdc-load-line-research-not-yet-tested-here) |
| AC/DC load line do nothing to delivered voltage in fixed mode | Only reported VID and package power move | Leave them; read VR VOUT, not Vcore | [owners' thread p.25](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-25) | [cpu/quirks.md](cpu/quirks.md#load-line-calibration-and-acdc-load-line-research-not-yet-tested-here) |
| Board Vcore sensor is socket-sense | Reads high under load | Compare with VR VOUT | [owners' thread p.27](https://www.overclock.net/threads/asrock-z790-nova-wifi-z790i-lightning-wifi-thread.1808124/page-27) | [cpu/quirks.md](cpu/quirks.md) |
| Power limits are useless as a "light-load boost" at fixed voltage | A limit drops about 12 bins, not 1 or 2 | One universal ratio | None found | [cpu/quirks.md](cpu/quirks.md#load-dependent-ratio-ideas-and-why-we-dropped-them) |
| 58x all-core is a silicon limit | More voltage only moves the crash later (2:11, 4:00, 6:48) | Stop adding voltage | None found | [cpu/results.md](cpu/results.md) |
| Ring 51 needs the voltage ceiling | Passes memory tests, fails Cinebench with GPU heat | Ring 50 | [Pakhtunov guide](https://pakt.cc/files/CPU_2.01_2026_EN_FINAL.pdf) (50x ceiling with HT) | [cpu/quirks.md](cpu/quirks.md#ring-51x-needs-the-voltage-ceiling-and-the-memory-test-cannot-see-it) |
| E-cores 47x hard-freeze | No bugcheck, no restart | 46x | None found | [cpu/quirks.md](cpu/quirks.md#e-cores-46x-passes-47x-freezes) |
| Errors are named on the core that saw them | A memory fault reported "on an E-core" | In-cache run to separate core from memory | None found | [ram/quirks.md](ram/quirks.md#errors-reported-on-a-core-are-not-always-the-core) |
| Silent crashes | No WHEA record at all | Count them as fails | None found | [cpu/quirks.md](cpu/quirks.md#errors-and-crashes-without-a-whea-record) |
| Cinebench resets its own affinity | `start /affinity` ignored | Job object affinity limit | None found | [cpu/quirks.md](cpu/quirks.md#cinebench-resets-its-own-affinity) |
| Liquid metal crusted in days | Hot P-cores, 19 C spread | Redo with a fresh syringe; compare C per 100 W first | None found | [cpu/thermals.md](cpu/thermals.md) |
| Cooling buys voltage before clocks | Repaste: about 10 C cooler, floor 20 mV lower, 58x still fails | Repaste before chasing voltage | None found | [cpu/results.md](cpu/results.md) |
| One 8-pin EPS at about 410 W | | Check the plug is fully seated | [Tom's Hardware review](https://www.tomshardware.com/pc-components/motherboards/asrock-z790i-lightning-wi-fi-review) | [cpu/quirks.md](cpu/quirks.md) |

## RAM (DDR5, Raptor Lake IMC)

| Quirk | Symptom | Remedy | Public | Details |
|---|---|---|---|---|
| tREFI is heat-bound | 65535 passes 60 short runs, fails TestMem5 at about 60 C | Keep 32767 until the sticks run cooler | [OCN temperature thread](https://www.overclock.net/threads/ddr5-most-temperature-sensitive-timings.1808660/) | [validation.md](validation.md#heat-is-part-of-the-test) |
| Looser tRFC does not fix a tREFI fault | 688 / 460 still failed | Change the refresh interval | None found | [ram/results.md](ram/results.md#the-heat-bound-fault-trefi) |
| The hotter stick sets the limit | Sticks 6 to 9 C apart | Log both; a fan under the DIMMs is worth 8 to 12 C | None found | [ram/quirks.md](ram/quirks.md#dimm-temperature-limits-for-24-gb-m-die) |
| tCL 36 trained, then corrupted boot files | Checksum errors loading Windows, recovery loops | Needs more voltage or cooling; failsafe cannot catch it | None found | [ram/quirks.md](ram/quirks.md#tcl-36-it-trained-and-then-it-corrupted-data) |
| tRDRD_sg 14 = no POST | No picture; five power cycles | 15 is the floor here | [Pakhtunov timings](https://pakt.cc/en/ddr5/timings/) (tCCD_L context) | [ram/quirks.md](ram/quirks.md#a-no-post-from-one-tertiary-trdrd_sg-14) |
| tWTR follows tWRRD | tWTR = tWRRD - tCWL - 6 here | Tune tWRRD; read tWTR back | OCN-INTEL / OCN-LIGHT posts; [Pakhtunov](https://pakt.cc/en/ddr5/formulas/) (constant 10) | [ram/quirks.md](ram/quirks.md#twtr-is-set-through-twrrd-and-it-moves-when-you-are-not-looking) |
| tRTP reads entered minus 1, not derived from tWR | NOT APPLIED read-backs | Expect the offset | None found | [scewin.md](scewin.md#6-read-back-offsets-and-derived-fields) |
| tRAS below tRCD + tRTP is not a hard floor | tRAS 68 and 64 passed below it | No gain below it, no harm seen | [integralfx](https://github.com/integralfx/MemTestHelper/blob/oc-guide/DDR4%20OC%20Guide.md) (rule) | [ram/quirks.md](ram/quirks.md#the-tras-floor-is-not-a-hard-limit) |
| tRFCpb matters on this BIOS | 796 to 560 was the single biggest latency step | Tune it; 420 floor here | OCN-INTEL post 32731 (100 ns) | [ram/quirks.md](ram/quirks.md#trfcpb-is-worth-a-look) |
| Training fallback signature | tRFC 395, DDR5-3600, tCL 29, TX 1.100 V | Any untouched value changed = NO TRAIN | None found | [ram/quirks.md](ram/quirks.md#the-trfc-395-fallback-signature) |
| WHEA id 1 after a failed training | "Fatal hardware error" with no core | Look for the fallback signature | None found | [validation.md](validation.md#whea-accounting) |
| DDR5 on-die ECC is invisible | Every RAM failure had WHEA 0 | Do not use WHEA as a RAM test | [ATP](https://www.atpinc.com/tw/blog/ddr5-what-is-on-die-ecc-how-is-it-different-to-traditional-ecc) | [ram/quirks.md](ram/quirks.md#ddr5-on-die-ecc-is-invisible-to-whea) |
| VDD_IMC is TX, VDD_CPU is VDD2 (ASRock names) | Auto follows XMP voltage (1.45 V) | Set both explicitly | Owners' threads | [ram/quirks.md](ram/quirks.md#asrock-bios-behaviours-we-hit) |
| DIMM sensors read 0 after a cold boot | No temperatures | Start HWiNFO once to initialise the SPD hubs | None found | [ram/quirks.md](ram/quirks.md#dimm-sensors-read-0-after-a-cold-boot) |
| 14th gen SA freeze bug | Hard freeze above a chip-specific VCCSA | Raise SA in small steps with a VT3 pass | [OCN SA thread](https://www.overclock.net/threads/14900k-14900k-cpu-sa-voltage-issue-freezing-problem-and-solved.1808469/) | [ram/quirks.md](ram/quirks.md#the-14th-gen-sa-bug) (not tested here) |
| Boot Failure Guard keeps the bad value | NVRAM intact after a no-POST | Export after any rough boot | None found | [ram/quirks.md](ram/quirks.md#asrock-bios-behaviours-we-hit) |
| ASRock Timing Optimization | May rewrite Auto timings | Set every timing you care about | None found | [ram/quirks.md](ram/quirks.md#asrock-bios-behaviours-we-hit) |

## SCEWIN and BIOS automation

| Quirk | Symptom | Remedy | Public | Details |
|---|---|---|---|---|
| Import locked on ASRock Z590+ | "Error in writing variable", "System configuration not modified" | BIOS admin password + `/cpwd` | [SCEHUB](https://github.com/ab3lkaizen/SCEHUB); [recipe](https://gist.github.com/Arie/af8e353dec6ce6b47359529571ace45f) | [scewin.md](scewin.md#1-import-is-locked-on-asrock-z590-and-newer) |
| Prompts embed the live value | Stale scripts silently skip questions | Fresh export before every write | None found (SCEHUB covers "Unmatched question" with other causes) | [scewin.md](scewin.md#2-every-prompt-contains-the-live-value) |
| Partial writes | Three fields written, the rest back to XMP | Write the full set, read back | None found | [scewin.md](scewin.md#3-write-the-full-set-every-time) |
| TcgStorageDynamicSetupVar error | On every import | Harmless if read-back is clean | None found for this variable | [scewin.md](scewin.md#4-harmless-noise) |
| Fields with no live label | Vcore, E ratio, VDD/VDDQ unprovable from an export | Verify in HWiNFO or by measured clock | None found | [scewin.md](scewin.md#5-value-label-and-what-the-bios-really-runs) |
| Fallback timings break a prepared restore | Failsafe v1 did nothing | Failsafe v2: fresh export at restore time | None found | [scewin.md](scewin.md#8-after-a-training-fallback-the-prompts-change) |
| Failsafe limits | No help for no-POST, freezes, or deaths in the first 18 s | Risky steps only with someone at the power button | None found | [scewin.md](scewin.md#the-startup-failsafe) |
| Saved OC profiles unreachable | Not in the export | Save them in the BIOS by hand | None found | [scewin.md](scewin.md#9-what-scewin-cannot-reach) |
| "Auto" Vcore on ASRock is adaptive 1.38 to 1.46 V | A recovery card said "Vcore back to Auto" | Recovery steps always name a number, mode Fixed | None found | [scewin.md](scewin.md#manual-path-for-the-person-at-the-keyboard) |

## Testing and validation

| Quirk | Symptom | Remedy | Public | Details |
|---|---|---|---|---|
| 10-minute floors | 30 mV optimistic | Confirm with 30 minutes | [CoreCycler](https://github.com/sp00n/corecycler) (long runs advised) | [validation.md](validation.md#run-length-10-minutes-overstates-the-floor) |
| Single passes near the edge | 10 passed, 30 failed | 1 / 3 / 10+ runs | [Rule of three](https://en.wikipedia.org/wiki/Rule_of_three_(statistics)) | [validation.md](validation.md#run-counts-and-what-zero-failures-proves) |
| Marginal base | Innocent steps fail | Control runs | [integralfx](https://github.com/integralfx/MemTestHelper/blob/oc-guide/DDR4%20OC%20Guide.md) (general method) | [validation.md](validation.md#control-runs) |
| y-cruncher launcher | Affinity on the wrong process | Mask the worker after 20 s | [Rynez README](https://github.com/exidum79/Rynez-Stability-Monitor) | [validation.md](validation.md#y-cruncher-pitfalls) |
| "Stop on Error: Enabled" | Parser logs a pass as FAIL | Match exact result lines | None found | [validation.md](validation.md#y-cruncher-pitfalls) |
| TestMem5 coverage | 40% when memory was not free | Free memory, read the coverage line | [CoolCmd TM5](https://github.com/CoolCmd/TestMem5) (allocation bug in 0.12) | [validation.md](validation.md#testmem5-pitfalls) |
| TM5 0.13.1 wording | Clean pass logged as FAIL | "Testing completed" = pass; "An error N occurred" = fail | None found | [validation.md](validation.md#testmem5-pitfalls) |
| y-cruncher misses refresh faults | 60 minutes clean, TM5 fails in 16 | Run both, hot | Reddit / OCN reports | [validation.md](validation.md#heat-is-part-of-the-test) |
| CPU heat inside a RAM test | 459 W hard reset | Cool CPU load, temperature guard | None found | [validation.md](validation.md#keep-the-cpu-out-of-a-ram-test) |
| WHEA from short windows | "0" when 37 in 30 days | Count over the whole period, map to windows | [OCN WHEA 19](https://www.overclock.net/threads/about-whea-event-19-warnings.1536052/) | [validation.md](validation.md#whea-accounting) |
| Crash dump on unstable RAM | Blue screen frozen about 1 h (0x80) | Dumps off during unattended sweeps only | [MS Q&A](https://learn.microsoft.com/en-us/answers/questions/3799557/blue-screen-of-death-not-generating-dump-file) (stuck at 0%) | [validation.md](validation.md#crash-dumps-on-or-off) |
| 0x9F shutdown hangs read as training | 323 to 353 s boots | Check for 1001 / 0x9F first; close GPU-heavy apps before restarts | [MS 0x9F](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/bug-check-0x9f--driver-power-state-failure); [NVIDIA forum](https://forums.developer.nvidia.com/t/bug-nvlddmkm-sys-0x9f-driver-power-state-failure-subcode-3-when-entering-s4-hibernate-d3-set-power-irp-never-completes-rtx-4060-windows-10-22/386712) | [validation.md](validation.md#slow-boots-0x9f-shutdown-hangs-not-memory-training) |
| Piped output kills runs | A broken pipe to another machine ended a 30-minute run | Mirror beside the run, never in its pipe | None found | [fafo-log.md](fafo-log.md) |
| Unflushed journal | Zero bytes after a reboot | fsync before every reboot | None found | [tools.md](tools.md#tooling-pitfalls-we-hit-so-you-do-not-have-to) |

## Windows, drivers, monitoring

| Quirk | Symptom | Remedy | Public | Details |
|---|---|---|---|---|
| WSL swap in Temp | Temp purge hits live swap | Exclude the live GUID folder | [Microsoft](https://learn.microsoft.com/en-us/windows/wsl/wsl-config) | [windows.md](windows.md#the-wsl-swap-file-lives-in-temp) |
| Bulk uninstallers | Start Menu wiped | Snapshot first | None found | [windows.md](windows.md#snapshot-the-start-menu-before-bulk-uninstalls) |
| Wildcards | `*Bst*` hit USBSTOR, `^MSI` hit msiserver | Select by identity | None found | [windows.md](windows.md#never-select-by-wildcard) |
| Old board-specific iGPU driver | HWiNFO hangs in `ze_intel_gpu64.dll` | Remove it; or HWiNFO oneAPI off | [HWiNFO forum](https://www.hwinfo.com/forum/threads/hwinfo64-8-60-6120-crashes-during-initialization-%E2%80%94-intel-level-zero-dll-asus-g16-core-ultra-9-185h.11389/) | [windows.md](windows.md#the-old-intel-igpu-driver-that-crashes-hwinfo) |
| Windows Update reinstalls it | Fix undone in days | `ExcludeWUDriversInQualityUpdate=1` | [Intel 000087834](https://www.intel.com/content/www/us/en/support/articles/000087834/graphics.html) | [windows.md](windows.md#the-old-intel-igpu-driver-that-crashes-hwinfo) |
| Monitor on the iGPU | About 5.5% lower GPU benchmark | All monitors on the card, iGPU off | [XMG](https://help.xmg.gg/hc/en-gb/articles/40377154604317-What-is-the-difference-between-iGPU-and-dGPU-connections-for-external-monitors) (hybrid copy) | [windows.md](windows.md#a-monitor-on-the-igpu-costs-gpu-performance) |
| Idle iGPU power | 0.008 W; disabling frees nothing | Disable for display reasons only | None found | [windows.md](windows.md#disabling-the-igpu-frees-no-measurable-cpu-power) |
| HWiNFO takes the Super I/O from a fan tool | Fans to about 3,000 rpm while it runs | Expect it, or close one tool | [HWiNFO forum](https://www.hwinfo.com/forum/threads/hwinfo64-taking-over-fancontrol-sensor-settings.8390/); [r/FanControl](https://www.reddit.com/r/FanControl/comments/1goa24j/fan_control_borked_by_hwinfo/) | [cpu/quirks.md](cpu/quirks.md#a-monitoring-tool-can-take-the-fans-away-from-the-fan-controller) |
| Two programs on one HID device | Stale replies, coolant reading lost | Flush the queue before each request | [Microsoft HidD_FlushQueue](https://learn.microsoft.com/en-us/windows-hardware/drivers/ddi/hidsdi/nf-hidsdi-hidd_flushqueue); [OSR](https://community.osr.com/t/theory-buffering-in-hid-class-driver/30591) | [cpu/quirks.md](cpu/quirks.md#two-programs-on-one-usb-hid-device-read-each-others-replies) |
| WSL page cache | VmmemWSL 23 GB | drop_caches; `autoMemoryReclaim` | [Microsoft](https://learn.microsoft.com/en-us/windows/wsl/wsl-config) | [windows.md](windows.md#wsl) |
| Game DVR off turns Game Mode off | | Re-check Game Mode | None found | [windows.md](windows.md#small-ones) |
