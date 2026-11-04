# ASUS ROG Strix OLED XG34WCDG: monitor control protocol reference

An independent description of how a host program can read and control the ASUS ROG Strix OLED XG34WCDG, a 34-inch
QD-OLED gaming monitor: the DDC/CI (MCCS) codes behind its on-screen menu, the USB functions behind its built-in hub,
and what ASUS's own software installs. It is meant as the reference for running this monitor without ASUS's software.

| Part | How the host reaches it | What it does | Document |
|---|---|---|---|
| DDC/CI | VESA DDC/CI over the video cable, through Windows' `dxva2.dll` | brightness and colour, GameVisual and colour space, HDR mode, panel care (OLED Care), the Neo Proximity Sensor, the Aura effect, the power indicator, identity | [asus-xg34wcdg.md](asus-xg34wcdg.md), sections 1-8 |
| Aura USB device | USB HID `0B05:1C2B` interface 1, behind the monitor's hub | per-LED lighting (Armoury Crate's path); documented from published sources and Windows' records only | [asus-xg34wcdg.md](asus-xg34wcdg.md), section 9 |
| ASUS software | the monitor's Windows driver package | DisplayWidget Center's bootstrapper, ASUS's command-line tool | [asus-xg34wcdg.md](asus-xg34wcdg.md), section 10 |

Coverage per code and change history: [STATUS.md](STATUS.md).

## Why this exists

ASUS publishes no VCP code table. Public maps of ASUS's newer manufacturer-code scheme come from community projects
written for other models, mainly the ROG Swift PG32UCDM3 and the ROG Strix XG27AQDMGR (see Credits), and ASUS's own
command-line tool names properties but no codes. On 2026-11-01 this monitor's capability string and every VCP value
were read with independently written third-party code, and the owner then changed one or two items in the monitor's
own menu at a time while the same code read every code before and after, 35 rounds in all, plus one reading with
Windows HDR on. Until 2026-11-04 the same code read the monitor every minute. This reference writes down what was read,
which published claims held on this model, and what is still unknown.

There was **no USB capture** of this monitor, and nothing was ever sent to its USB functions: the USB (Aura) path in
section 9 is documented from published sources and from the device properties Windows records, and is unverified on
this model. On the afternoon of 2026-11-04 the same third-party code sent 47 DDC/CI writes to the monitor, one setting
at a time with the owner approving each and restoring every value, so the writing rules of section 7 are observed on
this unit (sections 5 to 7 say which codes were written and what the monitor did).

New compared with what was public before:

- **The XG34WCDG's capability string** (57 codes), and that it reports ASUS's scheme version 2.17 both in code `0xEF`'s
  maximum field and as `mccs_ver(2.17)`, which is not an MCCS version.
- **The sibling map holds on this model.** Every code of the published PG32UCDM3 map that was exercised matched: the
  GameVisual low byte of `0xDC`, Shadow Boost `0xE5`, Blue Light Filter `0xE6`, the cleaning reminder `0xF8`, Screen Move
  `0xF9`, Auto Input Detection in `0xFC`, and VRR and seven panel-care bits in `0xFD`.
- **New mappings:** Display Color Space in `0xDC`'s high byte; the Neo Proximity Sensor in `0xED` (screen-off minutes and
  distance); the Aura effect and colour in `0xF2`; the power indicator in `0xFC` bit `0x0001`; Sniper in `0xFD` bit
  `0x0004`, which clears itself; VividPixel on the MCCS sharpness code `0x87`. Every bit of `0xFD`'s support mask
  `0x787D` is now named.
- **`0xE2` is the HDR-mode word on this model** (`0x0102` HDR10 Gaming, `0x0104` True Black 400), and OLED Anti-Flicker
  shows as `0x0104` with Windows HDR off.
- **Monitor behaviour over DDC/CI:** a locked control reads `0xFE` of `0xFE`, but brightness reads 100 instead;
  brightness, colour preset, gamma and Shadow Boost are stored per GameVisual mode; the Blue Light Filter leaves the
  colour preset at User 1; the MCCS change flags `0x02` and `0x52` never move; gamma sits in the high byte.
- **Windows behaviour:** dxva2 passes the maximum field of non-continuous codes through unchanged; a Windows service in
  session 0 sees no physical monitor; physical-monitor handles are destroyed around display-mode changes; reads were
  stable for days and across sleep.
- **DisplayWidget Center's bootstrapper** in the monitor's driver package, which Windows starts at driver installation
  and which downloads ASUS's software.
- **What the monitor does with writes** (2026-11-04): every mapped setting a host would change was written and read
  back; the monitor **silently ignores** (no error, old value on read-back) a write to a control its HDR interlock has
  locked, an Aura effect while Aura Sync is on, and Aura colour `00`; Static Aura starts red and the colour byte
  persists; a VRR change re-plugs the display and destroys the DDC/CI handle for about ten seconds; Logo Detection is
  writable while Global Dimming is off; the power button is a panel-off that keeps DDC/CI, the DisplayPort link and
  "power on" alive, and another selected input keeps DDC/CI alive too.

## Scope

- The DDC/CI codes, encodings, value sets and behaviour of this monitor as Windows exposes them; the USB functions as
  Windows records them and as published for sibling models; what ASUS's software installs.
- Not covered: the DDC/CI wire layer below Windows' API (I2C addresses, checksums; see VESA's DDC/CI standard), the
  video signal itself, Armoury Crate's and DisplayWidget Center's user interfaces, and firmware updates of the scaler or
  the Aura controller, which are deliberately **not** described beyond the warning not to trigger them.
- One unit was observed: firmware 101 by the reading of section 1.4, connected over DisplayPort to an NVIDIA graphics
  card on Windows 11, with no ASUS application installed. Other firmware versions and other ASUS models may differ.

## How to read the tags

Every statement carries one of these tags.

| Tag | Meaning |
|---|---|
| **OBSERVED** | seen on this unit through Windows' DDC/CI API by independently written third-party code (not ASUS's software), with the date: the read-only probe of 2026-11-01, the OSD-correlation rounds (the owner changed one OSD setting, the code read every VCP code before and after), the session test, the read-only polling of 2026-11-01 to 11-04, and the write session of 2026-11-04 (one `SetVCPFeature` per setting, read back and restored). Things the owner saw on the monitor itself are marked "OBSERVED by the owner" |
| **METADATA** | read from ASUS's installed software without running it: printable strings, version resources and .NET metadata of the executables in the monitor's driver package, and file names; also the monitor's EDID and capability string read as data |
| **PUBLISHED** | first published by an open-source project, by ASUS (user guide, product pages, FAQ, its command-line tool's documentation), by VESA (MCCS, DDC/CI) or in Microsoft documentation, which is named. Where this unit confirmed it, the statement says so |
| **HOST** | read on the observing PC without driving the device: Windows device properties, hardware ids, driver bindings, registry values, the driver package's INF |
| **INFERRED** | a conclusion drawn from the evidence, not read from any source; the reasoning is given |
| **UNKNOWN** | a gap, with a note on what would close it |

## Bench sessions

Times are local time of the observing PC (UTC-7). Every session up to the afternoon of 2026-11-04 only read: no DDC/CI
write, no capability request after the probe, no USB handle, no ASUS program run. The write session of 2026-11-04 sent
DDC/CI writes only (no USB, no ASUS program).

| Date and time | What was done |
|---|---|
| 2026-11-04 14:44 to 16:06 | the write session: 47 `SetVCPFeature` writes, one setting per write, each read back after 50 ms with every listed code re-read afterwards, the owner approving each write and watching the panel, the back light and the power LED; Windows HDR on except 15:20 to 15:30 and 16:05 to 16:07; every value restored (sections 5 to 7) |
| 2026-11-04 15:43 to 15:53 | reads while the owner switched the monitor off by its button (twice) and set it to an input with no signal (section 6.7) |
| 2026-10-21 | Windows device properties, registry, the driver package's INF and its two executables (strings and metadata), the EDID (HOST, METADATA) |
| 2026-11-01 18:57:19 to 18:57:31 | the probe: capability string, then current and maximum of every listed code (Windows HDR off, DDC/CI on in the OSD, unelevated, user session) |
| 2026-11-01 19:03:59 to 19:44:18 | 37 snapshots of the 52 readable codes: two baselines and 35 rounds of one or two OSD changes by the owner (Shadow Boost, Blue Light Filter, Color Temp., three GameVisual modes, Uniform Brightness, Screen Move, the cleaning reminder, Auto Input Detection, OLED Anti-Flicker, the three dimming controls, taskbar, logo and boundary detection, the proximity sensor's distance and timer, Aura Sync, Aura effect and colour, the power indicator, VRR, VividPixel, Display Color Space, Aspect Control, Sniper); the monitor ended as it started |
| 2026-11-01 19:47:08 and 19:48:15 | the same four reads from the user session and from session 0 (LocalSystem, the context of a Windows service) |
| 2026-11-01 20:41:30 | a 38th snapshot with Windows HDR on (the menu greyed out) |
| 2026-11-01 20:27 to 2026-11-04 | read-only polling of 26 codes every 60 s through a helper process in the user's session, across restarts and three overnight sleeps |

The probe's and the snapshots' raw outputs are not published (they also hold data about other devices); every value
that matters is quoted in the document.

## Sources

The facts in this reference come from these sources only:

1. Read-only DDC/CI requests (Capabilities Request once, Get VCP Feature) to this monitor through Windows' `dxva2.dll`,
   made by independently written code on the owner's own PC, and the owner's account of the OSD state for each round.
2. Windows' device properties, registry and driver store on the same PC, read without opening the devices; the
   monitor's EDID as stored by Windows.
3. String, version-resource and .NET-metadata listings of the two ASUS executables in the monitor's driver package
   (nothing run, decompiled or disassembled).
4. The open-source projects and the vendor and standards documents listed under Credits, for the PUBLISHED statements.

No vendor source code was used or seen, and no firmware was read. The only vendor strings reproduced are file, service
and property names, version strings, and the monitor's own capability string and description.

## Credits

Projects are linked to their repositories; the facts were taken from their source trees (default branch) and trackers as
of October and early November 2026.

- **[sumi](https://github.com/Jenari-Dev/sumi)** (an open DisplayWidget Center replacement for Linux, GPL-2.0)
  published the map of ASUS's newer VCP scheme as verified on a PG32UCDM3 (`crates/sumi-core/src/features.rs`): the
  GameVisual numbering in `0xDC`'s low byte, `0xE1` to `0xEC`, `0xF3` to `0xFB`, the `0xFC` Auto Input bit and the `0xFD`
  panel-care bits; the scheme test in `0xEF`'s maximum and the firmware in `0xFF`'s maximum
  (`crates/sumi-core/src/monitor.rs`); the read-modify-write and pixel-cleaning rules for `0xFD`; and the PG32UCDM3's
  capability string (`fixtures/pg32ucdm3-capabilities.txt`). Facts only; nothing of its code or text is used.
- **[pixel-clean-gui](https://github.com/ReverendRick/pixel-clean-gui)** published pixel cleaning through `0xFD` bit
  `0x0010` on a PG32UCDM, its support mask `0x7879`, and the monitor dropping off DDC/CI during a cleaning.
- **[plasma-asus-rog-pg-xg-monitor-support](https://github.com/nvaert1986/plasma-asus-rog-pg-xg-monitor-support)**
  (`plasma_asus_monitor/features.py`, written for an XG27AQDMGR) published the `0xE2` HDR-mode values, `0xEE` as ELMB,
  `0x5F` as Adjustable HDR (with a contrary note), the `0xED` and `0xF2` encodings, Aura off as `0x0000`, and the warning
  that writing `0x61` restarts the monitor.
- **[ddcutil](https://github.com/rockowitz/ddcutil)** (`src/vcp/vcp_feature_codes.c`, GPL-2.0, facts only) tabulates the
  MCCS meanings and value sets used here (colour presets, input sources, gamma encoding, mute, usage time, OSD
  language, power mode); its [performance notes](https://www.ddcutil.com/performance_options/) give the DDC/CI waits
  and retry counts.
- **[ddccontrol-db](https://github.com/ddccontrol/ddccontrol-db)** (`db/monitor/AUS3453.xml`, `AUS32AD.xml`,
  `AUS2722.xml`, `AUS28B1.xml`) published code maps and a value dump of four older ASUS models, which show the older
  layout and the readings `0xEF` 177 and `0xE9` 5150 of 25720 that recur here.
- **[OpenRGB](https://gitlab.com/CalcProgrammer1/OpenRGB)** (`Controllers/AsusMonitorController/`,
  `Controllers/AsusAuraUSBController/`) published the Aura protocol of sibling ASUS monitors (report `0xEC`, `EC B0`,
  `EC 35`, `EC 40 84`), its detection by interface and usage, and the mainboard family's opcodes; its issue #5690
  (PG34WCDN), merge requests !3539 and !3548 (ROG Aura Monitor Light Bar, LED counts, the vendor interface) and issues
  #1689, #3137, #3859 and #4139 (older ASUS monitors with the same product string).
- **[hypercolor](https://github.com/hyperb1iss/hypercolor)** (`crates/hypercolor-hal/src/drivers/asus/`) published an
  independent Aura USB implementation with its packet limits and timing.
- **ASUS** published the XG34WCDG user guide, specification and product pages, FAQs 1053180 (pixel cleaning) and
  1050223 (DisplayWidget Center), and the
  **[ASUS Display Control CLI](https://github.com/ASUS-Display/asus-display-control)** documentation
  (`cli/docs/CLI_REFERENCE.md`, `skills/asus-display-control/SKILL.md`). ASUS is the vendor whose hardware and software
  were observed; product, menu and feature names are ASUS's.
- **VESA** publishes the MCCS and DDC/CI standards; **Microsoft** documents the low-level monitor configuration API
  (`dxva2.dll`) and, in PowerToys' Power Display documentation, the risks of capability requests and code scans;
  **KDE** PowerDevil merge request 456 records the NVRAM-wear concern behind read-before-write.

Nothing from these projects (code, comments or prose) is copied here, only the facts they established; value tables
they already publish are stated with credit.

## Related references

The same PC's other devices are documented in the same way, each in its own repository:

- [msi-meg-z790-ace-protocol](https://github.com/immortalbob/msi-meg-z790-ace-protocol): MSI MEG Z790 ACE motherboard: Mystic Light, the NCT6687D fan and sensor controller, MSI software and BIOS
- [msi-coreliquid-s360-protocol](https://github.com/immortalbob/msi-coreliquid-s360-protocol): MSI MEG CoreLiquid S360 cooler: controller and LCD
- [lian-li-uni-fan-sl-v2-protocol](https://github.com/immortalbob/lian-li-uni-fan-sl-v2-protocol): Lian Li UNI FAN SL V2 and the UNI HUB SL V2 controller
- [lian-li-universal-screen-protocol](https://github.com/immortalbob/lian-li-universal-screen-protocol): Lian Li Universal Screen 8.8": LCD and LED ring
- [razer-peripherals-protocol](https://github.com/immortalbob/razer-peripherals-protocol): Razer DeathAdder V2, Huntsman V2, Kraken V3 Pro and Synapse 3
- [msi-optix-g27c2-protocol](https://github.com/immortalbob/msi-optix-g27c2-protocol): MSI Optix G27C2 monitor: DDC/CI and EDID
- [msi-rtx4090-gaming-x-trio-protocol](https://github.com/immortalbob/msi-rtx4090-gaming-x-trio-protocol): MSI GeForce RTX 4090 Gaming X Trio: NVML and NVAPI control on Windows

## Licence

The documents in this folder are released under the **Creative Commons Attribution 4.0 International licence
(CC BY 4.0)**: use, copy, translate and build on them for any purpose, with credit to this work. See `LICENSE`.
No scripts are published with this reference.

## How to observe more

Everything here was read with Windows' own API; no extra software is needed, but the free tools
[ddcutil](https://www.ddcutil.com/) (Linux) and NirSoft's ControlMyMonitor (Windows) do the same reads.

1. Turn DDC/CI on in the monitor's OSD (System Setup → OSD Setup → DDC/CI). Run from the signed-in user's session; a
   Windows service cannot see the monitor.
2. Read the capability string once and keep it. Read only the codes it lists, current **and** maximum.
3. For each OSD item, take a full reading, change that one item, close the OSD, read again and compare. Keep a list of
   what you changed and when, and put everything back at the end.
4. Never write `0x61`, the reset and OSD-key commands (`0x04`, `0x08`, `0xEC`, `0xEB`, `0xE8`) or `0xFD` with bit
   `0x0010` set, and never call "save current settings".

Still unmapped, in case you want to contribute: `0xE9`, `0xF0`, `0xF7`, `0xFA`, `0x5F`, `0x61`, five bits of `0xFC`,
the HDR modes with Windows HDR on, the GamePlus codes, KVM and the shortcut-key ids; the Aura USB device's report
descriptor; and a USB capture of Armoury Crate driving the Aura device. See [STATUS.md](STATUS.md).

## Safety

Apart from the OBSERVED statements, nothing here was tested; the writes of 2026-11-04 were made to one unit, one at a
time, each read back and restored, with a person watching the panel. Anyone writing to their own monitor does so at
their own risk. In particular:

- this is an OLED panel: settings that weaken its burn-in protection, and starting or interrupting a pixel cleaning,
  have lasting consequences; `0xFD` mixes panel-care toggles with the cleaning trigger in one word;
- writing `0x61` is reported to restart the monitor, and input, power, KVM, HDR-mode and PBP changes can leave a black
  screen or re-plug the display;
- each write may be stored in the monitor's own memory: write only on change, never in a loop;
- never open the Aura device's interface 0 or the Realtek HID behind the monitor's hub, and never trigger a firmware
  update.
