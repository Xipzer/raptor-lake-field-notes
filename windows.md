# Windows: storage, drivers, the iGPU, WSL

The system side of the same machine: a Windows 11 desktop with Hyper-V, VBS and WSL2 always on, tuned and cleaned by an agent working from WSL.
These are the lessons from an August storage clean-up, a November driver clean-up and the iGPU work, including our own mistakes.
Labels: **Measured** (our logs), **Source** (public, linked), **Estimate** (our reasoning).

## Quick reference

| Quirk | What happened | Remedy |
|---|---|---|
| [WSL swap lives in Temp](#the-wsl-swap-file-lives-in-temp) | A Temp purge hit the live WSL swap file | Find and exclude the live `swap.vhdx` before cleaning `%LOCALAPPDATA%\Temp` |
| [Bulk uninstallers wreck the Start Menu](#snapshot-the-start-menu-before-bulk-uninstalls) | Start Menu destroyed, rebuilt by hand | Snapshot Start Menu, ProgramData and the uninstall list first |
| [Wildcards hit system items](#never-select-by-wildcard) | `*Bst*` matched USBSTOR, `^MSI` matched msiserver | Select by identity, never by name pattern |
| [Live profiles look like junk](#check-for-live-processes-before-deleting) | A live IDE profile was deleted | Check processes and services before deleting |
| [Junctions inflate sizes](#measure-without-following-junctions) | A "14 GB" folder was junction recursion | Exclude reparse points when measuring |
| [Deleting through WSL is slow](#delete-natively) | 20+ minutes for what takes seconds natively | Measure in WSL, delete with Windows tools |
| [Driver clean-up side effects](#driver-clean-up) | The iGPU fell back to an old driver | Re-check display drivers afterwards |
| [Board-specific driver outranks newer](#the-old-intel-igpu-driver-that-crashes-hwinfo) | HWiNFO hangs at GPU detection | Remove the old package; block driver delivery through Windows Update |
| [Windows Update reinstalls it](#the-old-intel-igpu-driver-that-crashes-hwinfo) | The fix undone three days later | `ExcludeWUDriversInQualityUpdate=1` |
| [Mixed-adapter desktop](#a-monitor-on-the-igpu-costs-gpu-performance) | About 5.5% lower GPU benchmark | Put every monitor on the graphics card, disable the iGPU |
| [Idle iGPU power](#disabling-the-igpu-frees-no-measurable-cpu-power) | Disabling it freed nothing measurable | Disable it for the display reason only |
| [WSL page cache](#wsl) | VmmemWSL at 23 GB with 1.7 GB in use | Drop caches; `autoMemoryReclaim` |
| [Game DVR toggle](#small-ones) | Turning DVR off also turned Game Mode off | Re-check Game Mode after changing DVR |

## Storage clean-up (August)

Result: C: from 711 to 413 GB used (about 298 GB back), the WSL disk image from 184.6 to 50.1 GB. Four incidents on the way, each now a rule.

### The WSL swap file lives in Temp

- **Measured:** a purge of `%LOCALAPPDATA%\Temp` hit the swap file of the running WSL VM.
- **Source:** WSL's default swap file is `%Temp%\swap.vhdx`, in a per-session GUID folder ([Microsoft: .wslconfig `swapFile`](https://learn.microsoft.com/en-us/windows/wsl/wsl-config); [GreenGorych](https://greengorych.io/blog/keeping-system-clean-crash-dumps-and-orphaned-swaps/)). Old GUID folders from crashed sessions are orphans and safe to delete; the live one is not.
- **Remedy:** never glob-delete Temp on a WSL host. Find the live folder (newest, size matches the swap in `free -h`), re-identify it right before deleting, and skip it. Or move it out with `swapFile=` in `.wslconfig`.

### Snapshot the Start Menu before bulk uninstalls

- **Measured:** a run of vendor uninstallers left the Start Menu nearly empty. It was rebuilt by hand to 38 verified shortcuts.
- **Remedy:** before any batch uninstall, copy the Start Menu folders (user and ProgramData) and export the uninstall list (ARP). Afterwards count `.lnk` files, not folders.
- Shortcut targets must be real launchable programs or stub launchers (for example Squirrel's `Update.exe --processStart App.exe`), never a DisplayIcon path or a version-numbered folder that the next update removes.
- To refresh the Start Menu after edits: touch the shortcut times and restart `StartMenuExperienceHost` and `SearchHost`.
- Public: we found no single public write-up of this failure; it is a known risk of running many uninstallers in a row.

### Never select by wildcard

- **Measured:** a pattern `*Bst*` meant for an emulator also matched `USBSTOR` (the USB storage driver). A pattern `^MSI` meant for a board vendor's tools also matched `msiserver` (Windows Installer). Both were caught before damage.
- **Remedy:** enumerate by identity: publisher, executable path, service image path, package family name, extension ID. Show the list, then act on the list.

### Check for live processes before deleting

- **Measured:** an IDE's remote-development profile looked like an old version folder and was deleted while it was in use. It was restored.
- **Remedy:** before deleting anything under AppData or ProgramData, check running processes and service image paths for it.
- The same applies to killing processes: walk the parent chain first. We once ended a process that belonged to another session's tool, thinking it was ours.

### Measure without following junctions

- **Measured:** a game launcher's folder showed about 14 GB through junction recursion.
- **Remedy:** exclude reparse points when measuring.

### Delete natively

- Measuring from WSL (`du`, `find`) is fine. Deleting big trees through `/mnt/c` goes over the 9P file bridge and took 20+ minutes for what `rd /s /q` does in seconds.

### Other rules from the same audit

- Tier everything: browsers and keys are never auto-deleted; user data is inspected and cleaned with confirmation; caches are cleaned on sight. "Never delete" does not mean "never inspect".
- Before deleting any project folder, check version control: is it a git repo, does it have a remote, is anything uncommitted or unpushed. Keep anything that would be lost.
- Uninstallers: use the registered `QuietUninstallString` where there is one; `msiexec /x {GUID} /qn /norestart` (1605 = not installed); Inno Setup `unins000.exe /VERYSILENT /SUPPRESSMSGBOXES /NORESTART`. Wrap each in a timeout. Leave interactive uninstallers to the person.
- "Uninstalled but not gone": search Run keys, scheduled tasks, services, shell extensions whose DLL is missing, and file associations for the vendor name. Unregister shell extensions and restart Explorer before deleting their DLLs.
- Pagefile fixed at 16 GB. Hibernation is not available with the hypervisor on (check `powercfg /a`).

## Driver clean-up (14 November)

**Measured:** the driver store went from 216 packages to 97.

Method:

1. Restore point and a copy of every driver package first.
2. A list of every package with: in use, hardware gone, no device present. The owner approved it line by line.
3. `pnputil /delete-driver <oem.inf>` **without** `/force`, so Windows refuses anything still in use. It refused none.
4. Orphaned services (an audio-effects leftover, an uninstalled RGB app's driver still loading at boot, two old virtual gamepad bus drivers): registry keys exported, driver files moved aside, then deleted.
5. One restart, then every device re-checked.

What went wrong or surprised us:

- **The chipset installer floods the store.** Intel's chipset package was needed for two driverless devices (SMBus, SPI). Its installer added 103 name-only INF packages; 101 unused ones were removed again the same way.
- **The iGPU fell back to an old board-specific driver** after the clean-up (next section).
- Three UAC prompts were needed where one had been promised. Bundle elevated work into one script, launched once.
- A leftover startup entry from the previous motherboard's audio package still ran at logon. Check Run keys after removing a platform's drivers.

## The old Intel iGPU driver that crashes HWiNFO

- **Measured:** HWiNFO 8.60 hung at "Detecting GPUs", inside the Intel driver's Level Zero library `ze_intel_gpu64.dll`. The UHD 770 was on a July 2024 board-specific package (31.0.101.5592). Windows ranks a package that names the board's exact hardware IDs above a newer generic one, so after the clean-up the old one won.
- **Fix:** export the old package as a backup, remove it with `pnputil /delete-driver <inf> /uninstall` (no force). Windows used the newer 32.0.101.7092 already in the store, and HWiNFO started.
- **It came back:** three days later Windows Update installed the old package again. Fix repeated, then driver delivery through Windows Update blocked with `HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate` `ExcludeWUDriversInQualityUpdate = 1` (DWORD). Undo by deleting the value. Side effect: no driver of any kind arrives through Windows Update, so install GPU drivers yourself.
- **Source, crash:** HWiNFO forum threads show the same faulting module on other Intel iGPUs, fixed by a newer Intel driver (7092 named by one user) or by turning off HWiNFO's "oneAPI Support" under Settings > Safety ([HWiNFO forum, Nov 2026](https://www.hwinfo.com/forum/threads/hwinfo64-8-60-6120-crashes-during-initialization-%E2%80%94-intel-level-zero-dll-asus-g16-core-ultra-9-185h.11389/); [older thread](https://www.hwinfo.com/forum/threads/beta-631_4255-crashes-while-detecting-graphic-card.6690/)).
- **Source, reinstall:** Windows Update replacing a newer Intel graphics driver with an OEM one is documented by Intel ([Intel 000087834](https://www.intel.com/content/www/us/en/support/articles/000087834/graphics.html)) and MSI ([MSI FAQ](https://www.msi.com/faq/faq-8665)), and the same policy value is the usual fix ([Super User](https://superuser.com/questions/1844080/how-to-force-intel-graphics-driver-version-10-18-15-4279-not-to-be-updated)).
- When installing Intel's own package, opt out of its tray apps (Graphics Software, Driver & Support Assistant) if you do not want them running.

## A monitor on the iGPU costs GPU performance

- **Measured, 18 November (RTX 5090, 3DMark Time Spy GT2 at a fixed GPU curve):** with one of four monitors on the Intel iGPU, GT2 ran at about 153 fps. With the iGPU disabled in Device Manager it ran at 161 to 162 fps, back on 153 when re-enabled: **about 5.5%**. It was the same on the old and the new Intel driver.
- Part of the cause turned out to be the benchmark window itself: launched from the command line, 3DMark opened a 3840 x 2160 **window** that spanned two monitors, one of them on the iGPU, instead of going fullscreen. Frames then had to cross between adapters. The GPU-side details are in [blackwell-field-notes](https://github.com/Xipzer/blackwell-field-notes).
- **Fix:** all four monitors on the graphics card (the fourth through an HDMI-out to DisplayPort-in active adapter; active adapters only work in one direction), the iGPU disabled in Windows, and in the BIOS (ASRock: IGPU Multi-Monitor = Disabled, set through SCEWIN, token 96).
- **Source:** cross-adapter copies when the output GPU differs from the render GPU are documented for hybrid laptops ([XMG](https://help.xmg.gg/hc/en-gb/articles/40377154604317-What-is-the-difference-between-iGPU-and-dGPU-connections-for-external-monitors)), and desktop users report the same with a second monitor on the iGPU ([TechPowerUp forum](https://www.techpowerup.com/forums/threads/using-i-gpu-for-basics-and-gpu-for-software-and-gaming.323920/); [XDA](https://www.xda-developers.com/igpu-can-be-fantastic-addition-gaming-pc-even-with-dedicated-gpu/)). We found no public measurement for an RTX 5090.
- The fear that a high-refresh DSC monitor would use up the card's display heads did not apply: one 5120 x 1440 240 Hz DSC monitor and three 1080p monitors (360, 240 and 144 Hz) all run on the one card.

## Disabling the iGPU frees no measurable CPU power

- **Measured (HWiNFO, before and after, same GPU test running):** the idle iGPU drew about **0.008 W**. CPU package 155.8 against 152.8 W, rest-of-chip 1.14 against 1.13 W, Vcore 1.312 V both, SA VID 1.251 V both, DIMMs 40 / 41 against 38.5 / 39 C. All within run-to-run noise.
- So the iGPU is already power-gated when nothing uses it. Turning it off for power, heat or CPU voltage headroom gains nothing we could see. Its only effect here was the multi-adapter one above.
- Public: we found **no public measurement** of idle UHD 770 power on a desktop with a discrete card; this is ours.

## Screens off and the iGPU

The legacy `SC_MONITORPOWER` broadcast turned the monitors off, but the one on the iGPU came straight back, and some iGPU DisplayPort outputs stayed half awake ("No Signal" flashes).
Our tray app [blackout-tray](https://github.com/Xipzer/blackout-tray) switches screens off by setting the display idle timeout to 1 second instead, and powers listed monitors off over DDC/CI. With every monitor on the graphics card this matters less.

## WSL

- **VmmemWSL at 23 GB while Linux programs used 1.7 GB (Measured, 17 November).** The rest was Linux page cache from the test files. `wsl.exe -u root sh -c "echo 3 > /proc/sys/vm/drop_caches"` gave it back.
  `autoMemoryReclaim` in `.wslconfig` (values `gradual` or `dropCache`) makes it automatic ([Microsoft: .wslconfig](https://learn.microsoft.com/en-us/windows/wsl/wsl-config)). Free the memory before a memory tester sizes itself, or its coverage drops.
- **Compacting the disk image:** clear caches, `sudo fstrim -av`, `wsl --shutdown` from an elevated Windows prompt, confirm no `vmmem` process, then diskpart `select vdisk file=...`, `attach vdisk readonly`, `compact vdisk`, `detach vdisk`. If your agent lives inside WSL, this has to run as a detached elevated Windows process.
- **Measured:** two compaction runs collided and left the disk image attached. Use a mutex and always detach in a finally block. A stuck attach clears with diskpart `detach`, or by restarting the WSL service.
- `/tmp` in WSL is wiped when WSL restarts. Keep logs somewhere persistent.

## Small ones

- **Game DVR and Game Mode:** turning Game DVR off once also turned off `AutoGameModeEnabled`. Check Game Mode after changing DVR.
- **RAM lighting on sleep:** removing Kingston FURY CTRL looked harmless and was not: it is what blanks the DIMM lighting when the PC sleeps. It went back.
- **Autostart changes:** ask the person first. We turned off an autostart entry the owner wanted.
- **Monitors:** `Win32_DesktopMonitor` under-reports. Use `WmiMonitorID` plus each monitor's PnP parent to see which GPU drives which screen. Monitor order changes after restarts.
