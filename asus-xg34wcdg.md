# ASUS ROG Strix OLED XG34WCDG: DDC/CI and USB control protocol

Part of the [ASUS ROG Strix OLED XG34WCDG protocol reference](README.md). Tags (**OBSERVED**, **METADATA**,
**PUBLISHED**, **HOST**, **INFERRED**, **UNKNOWN**) and the bench sessions are explained there. Times are local time of
the observing PC (UTC-7). "The probe" is the read-only DDC/CI probe of 2026-11-01, 18:57:19 to 18:57:31; "the OSD
rounds" are the OSD-correlation rounds of the same evening, 19:03:59 to 19:44:18, plus one HDR snapshot at 20:41:30.
VCP values are written as 16-bit words, high byte first: `0x0102` has high byte `0x01` and low byte `0x02`. A reading
is given as "current of maximum".

Short version (each point is tagged in the sections below): almost everything a host can change on this monitor goes
over DDC/CI, the VESA monitor-control channel in the video cable, which Windows exposes through `dxva2.dll` to
processes in the signed-in user's session only. The monitor answers a 950-byte capability string listing 57 VCP codes
and speaks ASUS's newer manufacturer-code scheme, which it reports as version 2.17. Above `0xDF` most codes are ASUS's
own; several pack two settings into the two bytes of one code, and two (`0xFC`, `0xFD`) are bit fields. A control that
another setting has locked reads `0xFE` of `0xFE`. The monitor's USB hub carries an Aura lighting device (`0B05:1C2B`)
whose protocol is known only from sibling monitors; its effects are also reachable over DDC/CI (`0xF2`). No write was
ever sent to this unit: everything below about writing is guidance, not observation.

## 1. Device

### 1.1 The monitor

| Item | Value | Source |
|---|---|---|
| Panel | 34-inch QD-OLED, 1800R, 21:9, 3440 × 1440, 10-bit | **PUBLISHED**, ASUS specification page and user guide |
| Refresh | up to 175 Hz; variable refresh 48 to 175 Hz on DisplayPort and HDMI | **PUBLISHED**, same; the EDID range limits agree (HOST, section 1.3) |
| HDR | VESA DisplayHDR 400 True Black; OSD HDR modes Gaming HDR, Cinema HDR, Console HDR, DisplayHDR 400 True Black, Adjustable HDR, Dynamic Brightness Boost | **PUBLISHED**, ASUS user guide p. 3-5 |
| Inputs | DisplayPort 1.4 (DSC), two HDMI 2.1, USB-C (DisplayPort Alt mode, 15 W) | **PUBLISHED**, ASUS specification page |
| USB | a hub with USB-B and USB-C upstream, three downstream ports, KVM switching between the upstreams | **PUBLISHED**, ASUS user guide pp. 1-3 and 3-14 |
| Other | earphone output, no speakers; Neo Proximity Sensor beside the power indicator; Aura lighting on the rear cover and the stand's base cover | **PUBLISHED**, ASUS user guide pp. 1-4, 3-9, 3-11 |
| Panel care | Screen Saver (dimming), Pixel Cleaning (about 6 minutes, also automatic in standby), Screen Move, logo and taskbar detection, uniform brightness | **PUBLISHED**, ASUS user guide p. 3-8 and ASUS FAQ 1053180 |

The observed unit was connected to an NVIDIA graphics card over DisplayPort (HOST); input `0x60` read `0x0F`,
DisplayPort-1, throughout (OBSERVED, 2026-11-01 to 11-04). The PC had a second monitor, which is not part of this
reference.

### 1.2 The control paths

| Path | Carries | Status here |
|---|---|---|
| DDC/CI over the video cable (MCCS VCP codes), reached through `dxva2.dll` | brightness and colour, GameVisual, HDR mode, panel care, proximity sensor, Aura effect, power indicator, identity | read side OBSERVED code by code; no write sent |
| USB HID, interface 1 of `0B05:1C2B` ("ROG Gaming Display Aura Device") | per-LED Aura colours through ASUS's Armoury Crate, per the user guide | PUBLISHED for sibling monitors only; nothing sent (section 9) |
| USB vendor interface 0 of `0B05:1C2B` (WinUSB) | an ENE controller's vendor channel (INFERRED, section 9.3) | **do not use** |
| USB HID `0BDA:1100` behind the monitor's hub | UNKNOWN (section 9.3) | **do not use** |

### 1.3 Identity on Windows (HOST, read 2026-10-21 without opening the device)

| Item | Value |
|---|---|
| Display device | `DISPLAY\AUS3456`, hardware id `MONITOR\AUS3456`, Windows' inbox `monitor` driver service |
| ASUS driver package | `xg34wcdg.inf`, ASUSTeK, DriverVer 06/03/2025 1.0.0.0: two colour profiles (`ASUS_Display_DisplayP3.icm`, `ASUS_Display_sRGB.icm`), `MaxResolution 3440,1440`, `DPMS 1`, two `DisplayHdrLevel` values and the service `DisplayWidgetCenterInstallService` (section 10.1). It sets **no** `CapabilitiesString` override, so the capability string Windows returns is the monitor's own |
| EDID | 384 bytes: base block, CTA-861 revision 3 and DisplayID 1.2, all three checksums valid. Manufacturer `AUS`, product `0x3456`, EDID 1.4, digital 10 bits per colour, DisplayPort interface, 80 × 34 cm, gamma 2.2, name descriptor `XG34WCDG`, built in week 46 of 2024 |
| EDID serial | a per-unit serial descriptor is present (not reproduced here); the numeric serial field is 0 |
| EDID timings | range limits 48-175 Hz vertical and 1000 MHz pixel clock; preferred timing 3440 × 1440 at 59.97 Hz; DisplayID type I timings 3440 × 1440 at 174.96, 119.99 and 86.98 Hz and 2560 × 1440 at 174.79 and 59.95 Hz; CTA-861 adds 2560 × 1440 at 120 Hz |
| dxva2 description | `ASUS XG34WCDG` for the physical monitor (OBSERVED, every read) |

To match a dxva2 physical-monitor handle to its EDID, list the active monitor device under the GDI display with
`EnumDisplayDevices` (flag `EDD_GET_DEVICE_INTERFACE_NAME`), take the PnP instance from the interface path and read
`Device Parameters\EDID` under that instance in the registry (OBSERVED: independently written code did this on every
start from 2026-11-01 to 11-04; **PUBLISHED**, PowerToys' Power Display design document describes the same matching).
The EDID serial descriptor is a real per-unit string on this model, so it can key per-device state (INFERRED).

### 1.4 Scheme and firmware identity (OBSERVED, every read since 2026-11-01)

| Code | Reading | Meaning |
|---|---|---|
| `0xEF` | `0x00B1` of `0x0211` | the **maximum** holds ASUS's VCP scheme version: high byte `0x02` = major 2, the "new VCP" layout; low byte `0x11` = minor 17. So the scheme is **2.17** |
| `0xFF` | `0x0030` of `0x0065` | the maximum holds the firmware version, `0x65` = 101; the low byte of the current value is a panel id, `0x30` |

- **Source of the rule.** sumi reads the scheme from `0xEF`'s maximum and the firmware from `0xFF`'s maximum, and
  serves its ASUS map only when the scheme's major version is 2 (**PUBLISHED**, sumi `crates/sumi-core/src/monitor.rs`).
  On this unit both readings are stable and the map of section 5 holds, so the rule works here (OBSERVED).
- **"2.17" is not an MCCS version.** The capability string ends `mccs_ver(2.17)` (OBSERVED), the same digits as
  `0xEF`'s maximum read as major `0x02` and decimal minor `0x11` = 17. VESA's MCCS versions are numbered 2.0, 2.1,
  2.2 and 3.0, with no 2.17 (**PUBLISHED**, VESA), so ASUS writes its own scheme version into the MCCS-version field on
  this model (INFERRED). The monitor does not list `0xDF` (VCP version), so its real MCCS level is not reported
  (OBSERVED). On the sibling PG32UCDM3 the two disagree: `mccs_ver(2.2)` in the string and scheme 2.18 in `0xEF`
  (**PUBLISHED**, sumi fixture `fixtures/pg32ucdm3-capabilities.txt` and its notes), so do not take the scheme from
  `mccs_ver` on other models.
- `0xEF`'s current value `0x00B1` (177) has no known meaning; ddccontrol-db's dump of an ASUS TUF VG34VQ3B shows the
  same 177 (**PUBLISHED**, `db/monitor/AUS3453.xml`).
- Firmware 101 is sumi's reading of `0xFF` (**PUBLISHED** rule, INFERRED here); ASUS's OSD "Information" page shows a
  firmware version that was not compared (UNKNOWN).

## 2. Transport: DDC/CI through Windows

### 2.1 API calls

DDC/CI was reached only through Windows' low-level monitor configuration API (**PUBLISHED**, Microsoft documentation
"Using the Low-Level Monitor Configuration Functions"). The calls made by the third-party code on this PC (OBSERVED):

| Step | Call | Used for |
|---|---|---|
| 1 | `MonitorFromPoint` (probe) or `EnumDisplayMonitors` + `GetMonitorInfo` (later code) | the `HMONITOR` of each Windows display |
| 2 | `GetNumberOfPhysicalMonitorsFromHMONITOR`, `GetPhysicalMonitorsFromHMONITOR` | the physical-monitor handle and its description |
| 3 | `GetCapabilitiesStringLength`, `CapabilitiesRequestAndCapabilitiesReply` | the capability string, **once**, in the probe (section 3) |
| 4 | `GetVCPFeatureAndVCPFeatureReply` | current value, maximum and type of one code |
| 5 | `DestroyPhysicalMonitors` | release the handles |

`SetVCPFeature` and `SaveCurrentSettings` were never called for this monitor (OBSERVED from the third-party code's
logs, 2026-11-01 to 11-04, which record every set with its word; none names this monitor). Nothing needed elevation:
every read ran unelevated in the user's session (OBSERVED).

### 2.2 Session 0: a Windows service cannot see the monitor (OBSERVED 2026-11-01)

The same four reads (`0x10`, `0x12`, `0xEF`, `0xFF`) were run once in the interactive user session (19:47:08) and once
as LocalSystem in session 0, the context of a Windows service, through a one-off scheduled task (19:48:15):

| Context | Displays Windows lists | Physical monitors | Reads |
|---|---|---|---|
| user session, interactive | the real displays | this monitor, as `ASUS XG34WCDG` | all four answered: `0x10` 95 of 100, `0x12` 80 of 100, `0xEF` `0x00B1` of `0x0211`, `0xFF` `0x0030` of `0x0065` |
| LocalSystem, session 0 | one virtual display `WinDisc`, 1024 × 768 | none: `GetNumberOfPhysicalMonitorsFromHMONITOR` fails with `0xC02625E5` ("the specified GDI device did not have any monitors associated with it") | none |

A service that controls this monitor needs a helper process in the user's session (INFERRED). The third-party code
used one from 2026-11-01 20:27 on; after each of three overnight sleeps (2026-11-02, 11-03 and 11-04, about 04:07 to
04:27) the only failed pass was the one before that helper was back, and the next pass about a minute later read the
monitor normally (OBSERVED).

### 2.3 The maximum field of non-continuous codes

Microsoft calls the maximum value undefined for non-continuous codes (**PUBLISHED**,
`GetVCPFeatureAndVCPFeatureReply` reference). This monitor puts data there and dxva2 passes it through unchanged:
`0xFD` reads maximum `0x787D` and `0xFC` maximum `0x089F`, exactly the masks the capability string lists as
`FD(787D)` and `FC(089F)` (OBSERVED, the probe). So ASUS's identity codes (`0xEF`, `0xFF`) and support masks can be read
on Windows without the capability string.

### 2.4 Timing

- The probe and the OSD rounds paused 60 ms before every read; the probe read the capability string and 52 codes in
  12 s (OBSERVED).
- Later code kept at least 50 ms idle between the end of one command and the start of the next and read 26 codes per
  start (the two identity codes and 24 settings) in 1.9 to 3.4 s, including opening the monitor through the
  session helper (OBSERVED, 51 starts from 2026-11-01 to 11-04). A read takes tens of milliseconds; Microsoft gives
  about 40 ms (**PUBLISHED**, Microsoft documentation).
- From 2026-11-01 20:27 to 2026-11-04 the same code read the monitor every 60 s. Apart from the episodes in section
  2.5 and the sleeps of section 2.2, every pass succeeded; this monitor never returned an I2C transmit error
  (`0xC0262582`) in any read on record (OBSERVED).
- The DDC/CI specification has the host wait 40 to 200 ms between a command and its reply; ddcutil retries up to 4
  times for writes and 10 times for reads (**PUBLISHED**, ddcutil's performance notes). dxva2 does the waits itself
  (INFERRED from the call times).

### 2.5 Errors seen

| Code | Windows name | When | Tag |
|---|---|---|---|
| `0xC0262584` | `ERROR_GRAPHICS_DDCCI_VCP_NOT_SUPPORTED` | every read of `0xE8`, `0xEB` and `0xEC`, the write-only command codes (section 5.4) | OBSERVED, the probe |
| `0xC026258D` | `ERROR_GRAPHICS_MONITOR_NO_LONGER_EXISTS` ("the operating system asynchronously destroyed the monitor which corresponds to this handle") | seven passes between 2026-11-01 20:33 and 20:52; each time, closing the handle and taking a new one at the next pass (one to two minutes later) worked | OBSERVED |
| `0xC02625E5` | (session 0, section 2.2) | every call from session 0 | OBSERVED |

The `0xC026258D` episode fell in the period in which a browser played HDR video and Windows' HDR (advanced colour)
was found on for this display (section 6.5). Windows invalidates physical-monitor handles on display-mode changes, and
an HDR switch is one (INFERRED from the error text). Do not cache handles across display changes; on this error, take
a new handle (OBSERVED to work).

## 3. Capability string

### 3.1 As read (OBSERVED, the probe)

```
(prot(monitor)type(LCD)model(XG34WCDG)cmds(01 02 03 07 0C E3 F3)vcp(02 04 08 10 12 14(03 04 05 06 07 08 09 0B) 16 18
1A 52 59 5A 5B 5C 5D 5E 5F(0005) 60(0F 11 12 1A) 61(01 02) 62 72(50 64 78 8C A0) 87(00 0A 14 1E 28 32 3C 46 50 5A 64)
8A 8D(01 02) C0 CC(01 02 03 04 05 06 07 08 09 0A 0C 0D 11 12 14 1A 1E 1F 23 24 25 26 27) D6(01 05) DC(0000 0100 0200
01 02 04 05 06 07 08 09 0A) E1(00 01) E2(0000 0100 01 02 03 04) E3(00 07 08 09 0A 0B 0C 0D 0E 0F) E4(00 01 02 03 04
05) E5(00 01 02 03 04) E6 E7(00 01) E8(01 02 03 04 05 06 07 08) E9 EA(00 01 02) EB(00 01 02 03 04 05 06 07) EC(01)
ED(0500 0A00 0F00 00 01 02 03 FF) EE EF F0(00 01) F2(0000 0100 0200 0300 0400 0500 0600 00 01 02 03 04 05 06) F3(02 03
04) F4(00 01 02 03 04 05 06 07) F5 F6 F7(00 01 02 03) F8(00 02 04 08) F9(00 01 02 03) FA FB(0100 0200 0300 0400 0500
0600 0700 0800 0900 0A00 0B00 0C00 01 02 03 04 05 06 07 08 09 0A 0B 0C) FC(089F) FD(787D)
FF)mswhql(1)asset_eep(40)mccs_ver(2.17))
```

The line breaks are added here, at spaces; the string has none. Windows reported a length of 950 bytes: 949 characters
and the terminating zero (OBSERVED).

- The request answered at the first try and Windows stayed normal (OBSERVED). plasma-asus states that ASUS panels
  reject the capability request (**PUBLISHED**, plasma-asus-rog-pg-xg-monitor-support); that does not hold for this
  unit. A few monitors with malformed strings have crashed Windows on this request (**PUBLISHED**, PowerToys Power
  Display documentation): read it once, cache it, and do not read it in a loop.
- `type(LCD)` although the panel is OLED (OBSERVED); the PG32UCDM3 says the same (**PUBLISHED**, sumi fixture).
- `cmds(...)` lists the DDC/CI commands the monitor accepts; `0C` is Save Current Settings, `F3` the capabilities
  request and `E3` its reply (**PUBLISHED**, VESA DDC/CI command numbers as used by ddcutil and Microsoft's
  `SaveCurrentSettings`).
- `asset_eep(40)`, `mswhql(1)` (OBSERVED); their meaning was not explored.

### 3.2 Reading the value lists

- **Two-digit tokens** are plain values (or low-byte values) of the code (OBSERVED: `14(03 ... 0B)` matches the
  readings `0x05`, `0x06`, `0x0B`).
- **Four-digit tokens** carry a 16-bit word. In `DC`, `E2`, `ED`, `F2` and `FB` they list the values of the **high byte**
  (`0100` = high byte 1) next to the two-digit low-byte values (OBSERVED: the OSD rounds moved the high and low bytes of
  `0xDC`, `0xED` and `0xF2` independently, within exactly these lists). In `FC(089F)` and `FD(787D)` the token is a
  **bit mask** of the supported bits (OBSERVED, section 2.3). In `72(50 64 78 8C A0)` two-digit tokens are values of the
  **high** byte (OBSERVED: `0x72` reads `0x7800`).
- **`5F(0005)`** has one four-digit token while `0x5F` reads `0x0001` of `0x0005` (OBSERVED); whether it is a value
  list, a range or a mask is UNKNOWN.
- A parser that reads every token as one byte misreads `DC`, `E2`, `ED`, `F2`, `FB`, `FC`, `FD` and `72` (INFERRED).
  The PG32UCDM3's string has the same token forms (**PUBLISHED**, sumi fixture).

### 3.3 Compared with the sibling PG32UCDM3 (**PUBLISHED**, sumi fixture; OBSERVED for this unit)

| | XG34WCDG | PG32UCDM3 |
|---|---|---|
| `mccs_ver` | 2.17 (= the scheme) | 2.2 (scheme 2.18) |
| standard codes absent here | `0x05`, `0xAC`, `0xAE` (no refresh-rate readout), `0xB2`, `0xB6`, `0xC6`, `0xC8`, `0xDF` | listed |
| manufacturer codes | `0xE1` to `0xFD` and `0xFF`; `0xE0` absent; `0xF0` added | `0xE1` to `0xEF`, `0xF2` to `0xFD`, `0xFF` |
| `0xFD` mask | `0x787D` | `0x7879` |
| `ED` list | `0500 0A00 0F00 00 01 02 03 FF` | also `0000` |
| `EC` list | `01` only | `01 FF` |

## 4. Code inventory

All 57 listed codes, with the probe's reading (current of maximum; HDR off, Racing mode) and where their meaning comes
from. "OSD rounds" means the code changed with exactly the OSD item named during the rounds of 2026-11-01 (OBSERVED).
Detail per code follows in section 5.

| Code | Reading | Meaning on this unit | Source of the meaning |
|---|---|---|---|
| `0x02` | `0x0001` of `0x0002` | MCCS new control value | **PUBLISHED**, MCCS; never changed in any of the 38 snapshots (OBSERVED) |
| `0x04`, `0x08` | not read | MCCS restore factory defaults, restore colour defaults (write-only) | **PUBLISHED**, MCCS |
| `0x10` | 95 of 100 | brightness | **PUBLISHED**, MCCS; OSD rounds |
| `0x12` | 80 of 100 | contrast | **PUBLISHED**, MCCS |
| `0x14` | `0x05` of `0x0B` | colour preset (Color Temp.) | **PUBLISHED**, MCCS; OSD rounds |
| `0x16`, `0x18`, `0x1A` | 100 of 100 each | red, green, blue video gain | **PUBLISHED**, MCCS; `0x1A` moved with the Blue Light Filter (OBSERVED) |
| `0x52` | 0 of 255 | MCCS active control | **PUBLISHED**, MCCS; never changed (OBSERVED) |
| `0x59` to `0x5E` | 50 of 100 each | six-axis saturation | **PUBLISHED**, MCCS 2.2 range; not OSD-correlated |
| `0x5F` | 1 of 5 | UNKNOWN; possibly Adjustable HDR | **PUBLISHED** with a doubt, plasma-asus; went 1 → 0 when HDR came on (OBSERVED) |
| `0x60` | `0x0F` of `0x1A` | input source | **PUBLISHED**, MCCS and the ASUS CLI |
| `0x61` | 2 of 2 | UNKNOWN; **writing it restarts the monitor** | **PUBLISHED**, plasma-asus (hazard) |
| `0x62` | 51 of 100 | audio volume (earphone output) | **PUBLISHED**, MCCS |
| `0x72` | `0x7800` of `0xA000` | gamma, value in the high byte | **PUBLISHED**, MCCS encoding; OSD rounds |
| `0x87` | 50 of 100 | VividPixel (the MCCS sharpness code) | OSD rounds |
| `0x8A` | 50 of 100 | saturation | **PUBLISHED**, MCCS |
| `0x8D` | 2 of 2 | audio mute: 1 muted, 2 not muted | **PUBLISHED**, MCCS |
| `0xC0` | 1096 of 0 | display usage time, hours | **PUBLISHED**, MCCS; it counted up 1096 → 1098 during the evening (OBSERVED) |
| `0xCC` | 2 of 39 | OSD language (2 = English) | **PUBLISHED**, MCCS and the ASUS CLI |
| `0xD6` | 1 of 5 | power mode | **PUBLISHED**, MCCS |
| `0xDC` | `0x0005` of `0x020A` | low byte GameVisual, **high byte Display Color Space** | sumi for the low byte; OSD rounds for both |
| `0xE1` | 0 of 1 | Power Setting | **PUBLISHED**, sumi; not OSD-correlated |
| `0xE2` | `0x0000` of `0x0104` | the HDR-mode word | **PUBLISHED**, sumi and plasma-asus; OBSERVED values |
| `0xE3` | 0 of 15 | GamePlus crosshair | **PUBLISHED**, sumi; not OSD-correlated |
| `0xE4` | 0 of 5 | GamePlus timer | **PUBLISHED**, sumi; not OSD-correlated |
| `0xE5` | 0 of 4 | Shadow Boost | **PUBLISHED**, sumi; OSD rounds |
| `0xE6` | 0 of 4 | Blue Light Filter | **PUBLISHED**, sumi; OSD rounds |
| `0xE7` | 0 of 1 | GamePlus display alignment | **PUBLISHED**, sumi; not OSD-correlated |
| `0xE8` | not readable | GamePlus position nudge, write-only command | **PUBLISHED**, sumi |
| `0xE9` | `0x141E` of `0x6478` | UNKNOWN | |
| `0xEA` | 0 of 2 | GamePlus FPS counter | **PUBLISHED**, sumi; not OSD-correlated |
| `0xEB` | not readable | OSD key injection, write-only command | **PUBLISHED**, sumi |
| `0xEC` | not readable | reset, write-only command | **PUBLISHED**, sumi |
| `0xED` | `0x0502` of `0xFFFF` | **Neo Proximity Sensor**: high byte screen-off minutes, low byte distance | OSD rounds (new) |
| `0xEE` | `0x00FE` of `0x00FE` | locked; ELMB according to plasma-asus | **PUBLISHED**, plasma-asus; lock marker OBSERVED |
| `0xEF` | `0x00B1` of `0x0211` | scheme version in the maximum | section 1.4 |
| `0xF0` | 0 of 0 | UNKNOWN | |
| `0xF2` | `0x0001` of `0x0606` | **Aura**: low byte effect, high byte colour | OSD rounds (new) |
| `0xF3` | 2 of 4 | KVM | **PUBLISHED**, sumi; not OSD-correlated |
| `0xF4` | 0 of 7 | PIP/PBP mode | **PUBLISHED**, sumi; lock marker OBSERVED |
| `0xF5`, `0xF6` | 0 of 0 each | PIP/PBP sources, GameVisual per window | **PUBLISHED**, sumi; not OSD-correlated |
| `0xF7` | 0 of 0 | UNKNOWN | |
| `0xF8` | 8 of 8 | Pixel Cleaning reminder, hours | **PUBLISHED**, sumi; OSD rounds |
| `0xF9` | 2 of 3 | Screen Move | **PUBLISHED**, sumi; OSD rounds |
| `0xFA` | 0 of 3 | UNKNOWN; went 0 → 1 when HDR came on | OBSERVED |
| `0xFB` | `0x0102` of `0x0C0C` | the two OSD shortcut keys | **PUBLISHED**, sumi; not OSD-correlated |
| `0xFC` | `0x0811` of `0x089F` | bit field: **power indicator**, Auto Input Detection, five unnamed bits | sumi for one bit; OSD rounds |
| `0xFD` | `0x7829` of `0x787D` | bit field: VRR, **Sniper**, panel-care toggles, pixel-cleaning trigger | sumi for most bits; OSD rounds |
| `0xFF` | `0x0030` of `0x0065` | firmware version in the maximum, panel id | section 1.4 |

Every code returned type "set parameter" (`MC_SET_PARAMETER`), none "momentary" (OBSERVED, the probe).

## 5. Codes in detail

### 5.1 Standard MCCS codes

The meanings and encodings are VESA MCCS (**PUBLISHED**, as tabled in ddcutil `src/vcp/vcp_feature_codes.c`); the
values and behaviour are OBSERVED on this unit unless tagged otherwise.

- **`0x10` brightness, 0 to 100.** Stored per GameVisual mode (section 6.2). Reads 100 while Uniform Brightness, OLED
  Anti-Flicker or HDR is on (OBSERVED, section 6), not the lock marker.
- **`0x12` contrast, 0 to 100.** Read 80 throughout.
- **`0x14` colour preset.** Listed `03` to `09` and `0B`: 4000, 5000, 6500, 7500, 8200, 9300 and 10000 K and User 1
  (**PUBLISHED**, MCCS values), exactly the OSD's eight Color Temp. choices (**PUBLISHED**, ASUS user guide p. 3-7).
  Seen: `0x05` 6500 K, `0x06` 7500 K, `0x0B` User 1 (OBSERVED). Stored per GameVisual mode; the Blue Light Filter
  switches it to User 1 (section 6.3).
- **`0x16`, `0x18`, `0x1A` gains, 0 to 100.** Blue Light Filter level 1 lowered `0x1A` from 100 to 93 (`0x5D`) and level
  0 restored it (OBSERVED); the red and green gains did not move.
- **`0x02` and `0x52`.** In MCCS a monitor sets `0x02` to 2 and names the changed control in `0x52` when a user changes
  a setting (**PUBLISHED**, MCCS). On this unit `0x02` stayed 1 and `0x52` stayed 0 in all 38 snapshots
  (OBSERVED): change detection must poll the codes themselves.
- **`0x59` to `0x5E` six-axis saturation** read 50 of 100 (OBSERVED). MCCS puts the nominal value at 127 (**PUBLISHED**,
  MCCS); this monitor uses the OSD's 0-100 scale (INFERRED from the maxima). Not OSD-correlated.
- **`0x60` input.** Listed `0F` DisplayPort-1, `11` HDMI-1, `12` HDMI-2 (**PUBLISHED**, MCCS) and `1A` USB-C (**PUBLISHED**,
  ASUS Display Control CLI value 26; outside the MCCS table). Read `0x0F`. Switching to an input with no source blacks
  out the screen, and some monitors then stop answering DDC/CI until the input is changed by hand (**PUBLISHED**,
  PowerToys documentation).
- **`0x62` volume** read 51 of 100, and 50 at the HDR snapshot (OBSERVED; why it moved one step is UNKNOWN).
- **`0x72` gamma.** Listed `50 64 78 8C A0`. MCCS encodes gamma × 100 − 100 (**PUBLISHED**, MCCS), so the list is 1.8,
  2.0, 2.2, 2.4 and 2.6, the OSD's five choices (**PUBLISHED**, ASUS user guide p. 3-7). **The value sits in the high
  byte**: `0x7800` = 2.2 in Racing, `0x6400` = 2.0 in FPS (OBSERVED). An earlier ASUS LCD dump shows the value in the low
  byte (**PUBLISHED**, ddccontrol-db `AUS3453.xml`); the encoding differs between models.
- **`0x87` = VividPixel**, 0 to 100 in steps of 10 (listed `00` to `64` by 10s); the OSD's VividPixel 50 → 60 moved it
  `0x32` → `0x3C` (OBSERVED). The OSD has no item called Sharpness (**PUBLISHED**, ASUS user guide).
- **`0x8A` saturation** read 50; **`0x8D` mute** read 2, not muted (OBSERVED).
- **`0xC0` usage hours.** 1096 at 18:57, 1097 at 19:17, 1098 at 20:41 (OBSERVED): whole hours, maximum reads 0.
- **`0xCC` OSD language.** The list has 23 entries, the number of OSD languages (**PUBLISHED**, ASUS user guide p. 3-13),
  and ends with `26` and `27`, Persian and Indonesian in ASUS's CLI numbering, past the MCCS table's end at `0x25`
  (**PUBLISHED**, ASUS Display Control CLI; MCCS). Read 2, English.
- **`0xD6` power mode.** Listed `01` on and `05` display off (MCCS: write-only) (**PUBLISHED**, MCCS). Read 1. Some
  monitors do not wake by software after an off command (**PUBLISHED**, PowerToys documentation); untested here.
- **`0x04`, `0x08`**: restore factory or colour defaults (**PUBLISHED**, MCCS). Never sent; do not send (section 11).

### 5.2 `0xDC`: GameVisual (low byte) and Display Color Space (high byte)

| Byte | Values | Seen in the OSD rounds |
|---|---|---|
| low | `01` Cinema, `02` Scenery, `04` User, `05` Racing, `06` RTS/RPG, `07` FPS, `08` MOBA, `09` Night Vision, `0A` sRGB Calibration | Racing `0x05` → Cinema `0x01` (19:09:03) → FPS `0x07` (19:09:31) → Racing (19:10:01) |
| high | `00` Wide Gamut, `01` sRGB, `02` DCI-P3 | Wide Gamut → DCI-P3 `0x0205` (19:39:21) → sRGB `0x0105` (19:40:26) → Wide Gamut `0x0005` (19:42:05) |

- The low-byte numbering is sumi's (**PUBLISHED**); the three values seen match it (OBSERVED), and the list holds nine
  values for the OSD's nine GameVisual modes (**PUBLISHED**, ASUS user guide p. 1-5), with sRGB Calibration `0A` in place
  of sumi's sRGB `03` as on the sibling (INFERRED for the six unseen values).
- **The high byte is new.** The capability string lists `0000 0100 0200` (OBSERVED); no public map names it.
- Changing GameVisual restores that mode's brightness, colour preset, gamma and Shadow Boost (section 6.2).
- A write must change one byte and keep the other as read (INFERRED from the packing; sumi writes the low byte and keeps
  the high byte, **PUBLISHED**).

### 5.3 `0xE2`: the HDR-mode word

| Reading | When | Tag |
|---|---|---|
| `0x0000` of `0x0104` | Windows HDR off, OLED Anti-Flicker off | OBSERVED, the probe and the OSD rounds |
| `0x0104` | Windows HDR off, OLED Anti-Flicker set to High (19:16:22) and then Middle (19:17:14): the same value for both levels | OBSERVED |
| `0x0102` | Windows HDR on for this display (advanced colour, `RGB_FULL_G2084_NONE_P2020`, 175 Hz, read with dxdiag), OSD HDR mode not recorded | OBSERVED 20:41:30 |
| `0x0104` | Windows HDR on, with DisplayHDR 400 True Black chosen in the OSD (the owner's account) | OBSERVED 2026-11-01 21:09 |

- **Encoding.** Low byte: `01` Cinema, `02` Gaming, `03` Console, `04` True Black 400 (**PUBLISHED**, sumi and
  plasma-asus agree); the capability string lists exactly `01` to `04` (OBSERVED). High byte: `01` is HDR10 in sumi,
  which also has `02` Dolby Vision; plasma-asus calls it a constant that rides along (**PUBLISHED**). This unit lists
  `0000 0100` only (OBSERVED), so the high byte is 0 (SDR) or 1 here; both readings of it fit.
- sumi treats any value other than 0 and `0xFE` as "HDR active" and states that changing the high byte re-plugs the
  display (**PUBLISHED**); untested here.
- The OSD's other HDR modes, Adjustable HDR and Dynamic Brightness Boost (**PUBLISHED**, ASUS user guide p. 3-5), have no
  value in the list (OBSERVED). plasma-asus names `0x5F` as Adjustable HDR on/off, while another comment in the same
  file calls it dead on its monitor (**PUBLISHED**); here `0x5F` went 1 → 0 when HDR came on (OBSERVED). UNKNOWN.
- **OLED Anti-Flicker shows as `0x0104`**, the True Black 400 value, and locks exactly the controls HDR locks (section
  6.4). That Anti-Flicker runs the panel in that HDR mode is INFERRED; plasma-asus lists Anti-Flicker as having no DDC
  code on its model (**PUBLISHED**). The Anti-Flicker level (Middle or High) cannot be read (OBSERVED).
- `0xE2` was at first taken for an Anti-Flicker register; the HDR reading of 20:41 corrected that (OBSERVED).

### 5.4 `0xE1` to `0xEC`

| Code | Meaning | Values | Status |
|---|---|---|---|
| `0xE1` | Power Setting | `00` Standard (OSD "Performance"), `01` Power Saving, which locks most other controls | **PUBLISHED**, sumi; list `00 01` OBSERVED; not correlated |
| `0xE3` | GamePlus crosshair | `00` off, `07` to `0F` crosshair styles (sumi: `01`-`06` older styles, `07`-`0F` newer) | **PUBLISHED**, sumi; this unit lists only `00` and `07`-`0F` (OBSERVED) |
| `0xE4` | GamePlus timer | `00` off, `01`-`05` = 30, 40, 50, 60, 90 minutes | **PUBLISHED**, sumi; list `00`-`05` OBSERVED |
| `0xE5` | Shadow Boost | `00` off, `01`-`03` levels, `04` Dynamic | **PUBLISHED**, sumi; Off → 1 seen (19:05:19); stored per GameVisual mode (FPS 3) |
| `0xE6` | Blue Light Filter | `00` off, `01`-`04` levels (no sub-list in the string; maximum 4) | **PUBLISHED**, sumi and ddccontrol-db for older ASUS models; 0 → 1 seen (19:07:03); side effects in section 6.3 |
| `0xE7` | GamePlus display alignment | `00`/`01` | **PUBLISHED**, sumi; not correlated |
| `0xE8` | GamePlus position nudge, **write-only command** | sumi: `01` up, `02` down, `03` right, `04` left; this unit lists `01`-`08` | **PUBLISHED**, sumi; values `05`-`08` UNKNOWN; reads fail with `0xC0262584` (OBSERVED) |
| `0xE9` | UNKNOWN | `0x141E` of `0x6478`: as two bytes 20 of 100 and 30 of 120 | the same reading appears in ddccontrol-db's VG34VQ3B dump (**PUBLISHED**); OSD transparency and timeout are an untested guess (INFERRED, weak) |
| `0xEA` | GamePlus FPS counter | `00` off, `01` number, `02` bar graph | **PUBLISHED**, sumi; not correlated |
| `0xEB` | OSD key injection, **write-only command** | `00` close, `01` show, `02`-`05` arrows, `06` enter, `07` back | **PUBLISHED**, sumi and the ASUS CLI's EZOSD values; reads fail (OBSERVED) |
| `0xEC` | reset, **write-only command** | `01` reset the current mode (sumi also has `FF` reset all; this unit lists only `01`) | **PUBLISHED**, sumi; reads fail (OBSERVED) |

On older ASUS models `0xE9` and `0xEB` are key locks and `0xEF` is Shadow Boost (**PUBLISHED**, ddccontrol-db
`AUS2722.xml`, `AUS28B1.xml`). Check `0xEF`'s maximum (section 1.4) before applying any ASUS map.

### 5.5 `0xED`: Neo Proximity Sensor (new)

| Byte | Meaning | Values |
|---|---|---|
| high | screen-off time, minutes | `05`, `0A`, `0F` (5, 10, 15); the OSD offers 5, 10 and 15 (**PUBLISHED**, ASUS user guide p. 3-9) |
| low | detection distance | `01` = 120 cm, `02` = 90 cm, `03` = 60 cm (OBSERVED); `00` and `FF` listed but not selected in the rounds |

OSD rounds (OBSERVED): 90 cm, 5 min `0x0502` → 60 cm `0x0503` (19:27:53) → 120 cm, 10 min `0x0A01` (19:28:50) → back
to `0x0502` (19:29:50). plasma-asus publishes the same encoding (**PUBLISHED**). The user guide offers a Tailored
mode besides the three distances (**PUBLISHED**, p. 3-9), so `FF` = Tailored is INFERRED; `00` = sensor off is
INFERRED.

At the HDR snapshot `0xED` read `0x0F00`, 15 minutes with low byte 0 (OBSERVED, 20:41:30). Whether the owner changed the
sensor between 19:44 and 20:41 or HDR changed it is UNKNOWN. Whether the sensor's presence state can be read is
UNKNOWN (not tested). The maximum reads `0xFFFF` (OBSERVED).

### 5.6 `0xEE`, `0xF0`, `0xF7`, `0xFA`

- **`0xEE`** has read the lock marker `0x00FE` of `0x00FE` in every read since the probe (OBSERVED). plasma-asus names it
  ELMB (Motion Sync) on/off (**PUBLISHED**, not marked verified there). ELMB is greyed out in the OSD on this unit even
  with VRR off (OBSERVED by the owner); ASUS's guide allows ELMB only at 87 Hz with VRR off (**PUBLISHED**, p. 3-2), and
  the unit ran at 175 Hz. So `0xEE` = ELMB, locked at this refresh rate, is INFERRED.
- **`0xF0`** (listed `00 01`, reads 0 of 0), **`0xF7`** (listed `00`-`03`, reads 0 of 0) and **`0xFA`** (no list, 0 of 3,
  1 under HDR) are UNKNOWN (OBSERVED readings). No public source names them.

### 5.7 `0xF2`: Aura lighting (new)

| Byte | Meaning | Values |
|---|---|---|
| low | effect | `00` Off, `01` Aura Sync, `02` Rainbow, `03` Color Cycle, `04` Static, `05` Breathing, `06` Strobing |
| high | colour, for Static, Breathing and Strobing | `01` red, `02` green, `03` blue, `04` cyan, `05` magenta, `06` yellow; `00` when no colour applies |

OSD rounds (OBSERVED): Aura Sync on `0x0001` → off `0x0000` (19:29:50) → an effect `0x0002` (Rainbow, 19:30:24) →
Static red `0x0104` (19:32:41) → Static blue `0x0304` (19:33:20) → Aura Sync on again `0x0001` (19:34:15).

- Values `00`, `01`, `02`, `04` and colours `01`, `03` were seen; the rest follow the order of ASUS's CLI values
  (`AuraMode` 0 OFF, 1 Aura Sync, 2 Rainbow, 3 Color Cycle, 4 Static, 5 Breathing, 6 Strobing; `AuraColor` 1 Red to
  6 Yellow, **PUBLISHED**, ASUS Display Control CLI reference) and the capability list `00`-`06` in both bytes (OBSERVED),
  so they are INFERRED.
- The OSD offers an effect only with Aura Sync off (OBSERVED; **PUBLISHED**, ASUS user guide p. 3-11). Leaving Aura Sync
  set the low byte to `00`.
- plasma-asus states that Aura off must be written as the whole word `0x0000` (**PUBLISHED**): clear the colour byte too.
- This gives the stored Aura effects without the USB path (section 9). With Aura Sync on, what the LEDs show is decided
  by USB (Armoury Crate) per the user guide (**PUBLISHED**, p. 3-16); with no USB host, what Aura Sync shows was not
  recorded (UNKNOWN).

### 5.8 `0xF3` to `0xFB`

| Code | Meaning | Values | Status |
|---|---|---|---|
| `0xF3` | KVM / USB upstream | sumi: `01` Smart KVM, `02` USB upstream B, `03` USB upstream C, `04` Auto KVM; this unit lists `02 03 04`, reads 2 | **PUBLISHED**, sumi (not exercised by sumi on hardware); not correlated. A report says `0xF3` changes show in the OSD of a PG32UCDM but do not move the hub (**PUBLISHED**, ROG forum, read only as a search summary) |
| `0xF4` | PIP/PBP mode | `00` off, `01`-`04` PIP corners, `05` PBP 50/50, `06`-`07` PBP wide; PBP re-plugs the display (sumi) | **PUBLISHED**, sumi; locked (`0xFE`) under Anti-Flicker and HDR (OBSERVED) |
| `0xF5`, `0xF6` | PIP sources (window A low byte, B high byte, input codes as `0x60`), GameVisual per window | | **PUBLISHED**, sumi; read 0 of 0 with PIP off (OBSERVED) |
| `0xF8` | Pixel Cleaning reminder, hours | `00` off, `02`, `04`, `08` | **PUBLISHED**, sumi; 8 → 4 → 8 seen (19:13:38, 19:15:17) |
| `0xF9` | Screen Move | `00` off, `01` light, `02` middle, `03` strong | **PUBLISHED**, sumi; Middle → Light → Middle seen (19:12:53, 19:13:38) |
| `0xFB` | OSD shortcut keys | high byte shortcut 1, low byte shortcut 2, function ids `01`-`0C` | **PUBLISHED**, sumi; reads `0x0102`; the id-to-function table for this model is UNKNOWN (the guide lists 15 shortcut functions, the string 12 ids) |

### 5.9 `0xFC`: bit field (mask `0x089F`)

| Bit | Meaning | Seen |
|---|---|---|
| `0x0001` | **Power Indicator** (set = on) | off `0x0810` (19:34:15), on `0x0811` (19:35:55) (OBSERVED, new) |
| `0x0010` | Auto Input Detection (set = on) | off `0x0801` (19:15:17), on `0x0811` (19:16:22) (OBSERVED; **PUBLISHED**, sumi) |
| `0x0002`, `0x0004`, `0x0008`, `0x0080` | UNKNOWN | clear throughout |
| `0x0800` | UNKNOWN | set throughout |

### 5.10 `0xFD`: bit field (mask `0x787D`)

| Bit | Meaning | Seen in the OSD rounds (OBSERVED) | Source of the name |
|---|---|---|---|
| `0x0001` | variable refresh rate (VRR) | off `0x7828` (19:35:55), on `0x7829` (19:38:21) | sumi; confirmed |
| `0x0004` | **Sniper** (GamePlus zoom) on | 1.2x `0x782D` (19:43:04); **cleared by itself** before the next read (19:44:18), and the owner found Sniper does not stay on | new |
| `0x0008` | Screen Dimming Control | off `0x7821` (19:19:59), on again (19:20:35) | sumi; confirmed |
| `0x0010` | **pixel cleaning running / start trigger** | clear throughout; not exercised | **PUBLISHED**, sumi and pixel-clean-gui, on PG32UCDM3 and PG32UCDM only |
| `0x0020` | Logo Detection (Auto Logo Brightness) | off `0x3009` together with Global Dimming off (19:24:19) | sumi; confirmed |
| `0x0040` | Uniform Brightness | on `0x7869` (19:10:45), off `0x7829` (19:11:27) | sumi; confirmed |
| `0x0800` | Taskbar Detection | off `0x7029` (19:22:54) | sumi; confirmed |
| `0x1000` | Boundary Detection | off `0x2829` (19:24:54) | sumi; confirmed |
| `0x2000` | Outer Dimming Control | off `0x5829` (19:20:35) | sumi; confirmed |
| `0x4000` | Global Dimming Control | off `0x3829` (19:21:11) | sumi; confirmed |

- Every bit of the mask is now named. The sibling's `0x0002` (ASCR, an LCD feature) is absent and `0x0004` added
  (OBSERVED; **PUBLISHED**, sumi for the sibling mask `0x7879`).
- **Global Dimming on locks Logo Detection on**: the OSD greys out Logo Detection while Global Dimming is on, so Logo
  Detection could be turned off only together with Global Dimming (OBSERVED by the owner; **PUBLISHED**, ASUS user guide
  p. 3-8).
- The three dimming bits belong to the OSD's Screen Saver submenu (OBSERVED by the owner).
- At the probe the word was `0x7829`: VRR, Screen Dimming, Logo, Taskbar, Boundary, Outer and Global Dimming on; Sniper,
  pixel cleaning and Uniform Brightness off (OBSERVED).
- **The pixel-cleaning bit is an action, not a setting.** On the siblings, setting it starts a cleaning of about six
  minutes, it reads 1 while the cleaning runs, and the monitor drops off DDC/CI during the run (**PUBLISHED**, sumi and
  pixel-clean-gui). On this unit it is UNKNOWN whether the bit behaves the same. A read-modify-write of `0xFD` that
  carries a set `0x0010` back would start a cleaning (INFERRED).

### 5.11 OSD items with no DDC code, and items not exercised

- **No code changed** (OBSERVED, the OSD rounds): the OLED Anti-Flicker level (Middle and High read the same), Aspect
  Control 21:9 Full → Pixel by pixel (at the native 3440 × 1440 this may change nothing on screen, INFERRED), and the
  Sniper zoom factor.
- **Greyed out:** ELMB, even with VRR off (OBSERVED by the owner; section 5.6).
- **Not exercised** (codes from sumi or none): KVM (`0xF3`), the OSD's HDR Setting in SDR, PIP/PBP (`0xF4`-`0xF6`),
  the other six GameVisual modes, crosshair (`0xE3`), timer (`0xE4`), FPS counter (`0xEA`), display alignment
  (`0xE7`), Power Setting (`0xE1`), the shortcut keys (`0xFB`), Virtual Screen, Customized Setting load and save, USB-C
  bandwidth and hub-in-standby, DisplayPort stream version and DSC, pixel cleaning, and the codes `0xE9`, `0xF0`,
  `0xF7`, `0xFA`.

## 6. Behaviour seen over DDC/CI

### 6.1 The lock marker

A control that another setting locks reads **current `0xFE`, maximum `0xFE`**, and its normal value and range come back
when the lock goes (OBSERVED):

| Locking condition | Codes at `0xFE`/`0xFE` | Other effects |
|---|---|---|
| Uniform Brightness on | `0xE6` | `0x10` reads 100 |
| OLED Anti-Flicker Middle or High | `0xE5`, `0xE6`, `0xF4` | `0x10` reads 100; `0xE2` = `0x0104` |
| Windows HDR on | `0xE5`, `0xE6`, `0xF4` | `0x10` reads 100; `0xE2` = `0x0102`; `0x5F` 1 → 0; `0xFA` 0 → 1 |
| always, so far | `0xEE` | (section 5.6) |

- Treat `0xFE`/`0xFE` as "locked", never as a level (INFERRED: `0xFE` is outside every range listed for these codes).
- Brightness does **not** show the marker under these locks: it reads 100 of 100. A brightness write in HDR would pass a
  marker check; whether the monitor refuses or ignores it is UNKNOWN (no write sent).
- The OSD greys out far more items in HDR than the three codes that show the marker: GameVisual, Shadow Boost,
  brightness, contrast, colour temperature, saturation, gamma and others (**PUBLISHED**, ASUS user guide p. 3-5; OBSERVED
  by the owner). Over DDC/CI, contrast (`0x12`), colour preset (`0x14`), gamma (`0x72`), saturation and GameVisual
  (`0xDC`) kept reading their SDR values in HDR (OBSERVED, the HDR snapshot). Whether writes to them are refused,
  ignored or stored for SDR is UNKNOWN.

### 6.2 Settings stored per GameVisual mode (OBSERVED, 19:09 to 19:10)

| Mode (`0xDC` low byte) | `0x10` brightness | `0x14` colour preset | `0x72` gamma | `0xE5` Shadow Boost |
|---|---|---|---|---|
| Racing (`05`) | 95 | `05` 6500 K | `0x7800` 2.2 | 0 |
| Cinema (`01`) | 90 | `06` 7500 K | `0x7800` 2.2 | 0 |
| FPS (`07`) | 100 | `05` 6500 K | `0x6400` 2.0 | 3 |

A GameVisual change rewrites these four codes from the mode's stored values; back in Racing the original values
returned. A value written to one of them belongs to the current mode only (INFERRED). Re-read all four after a
GameVisual change.

### 6.3 Blue Light Filter side effects (OBSERVED, 19:07 to 19:08)

Turning the filter from 0 to 1 also switched `0x14` from `0x05` (6500 K) to `0x0B` (User 1) and lowered the blue gain
`0x1A` from 100 to 93. Turning it back to 0 restored `0x1A` to 100 but **left `0x14` at User 1**; the owner set 6500 K
again by hand (19:08:32). ASUS's guide says activating the filter imports the Racing mode's defaults (**PUBLISHED**, p.
3-6). Record `0x14` before the filter is turned on if it should be restored afterwards (INFERRED).

### 6.4 OLED Anti-Flicker

Anti-Flicker (OSD: Off, Middle, High) has no code of its own (OBSERVED). While it is on, `0xE2` reads `0x0104` and the
monitor shows exactly the HDR lock pattern of section 6.1 (OBSERVED). ASUS describes the feature as stabilising the
refresh rate within a range, on DisplayPort and USB-C only (**PUBLISHED**, user guide p. 3-4). Reading `0xE2` = `0x0104`
with Windows HDR off therefore means "Anti-Flicker on" (INFERRED); with Windows HDR on it means True Black 400
(OBSERVED, section 5.3), and the two cannot be told apart from `0xE2` alone (INFERRED).

### 6.5 Windows HDR

At 20:41:30 the owner found most OSD items greyed out; dxdiag showed advanced colour (HDR) enabled for this display at
175 Hz, while a browser was playing HDR video (OBSERVED). The readings are in sections 5.3 and 6.1. The OSD greying is
the monitor's own HDR interlock (**PUBLISHED**, ASUS user guide p. 3-5); nothing had been written to the monitor
(OBSERVED). Whether the Windows setting or the browser's video turned the HDR signal on cannot be separated from the
evidence (UNKNOWN).

### 6.6 Stability

- Two complete reads 6 s apart were identical; across the 35 change rounds no code changed without an OSD action except
  `0xC0`, the hour counter (OBSERVED). The final round returned every code to the baseline apart from `0xC0` (OBSERVED).
- Polling every 60 s for two and a half days, across three sleeps and many restarts of the reading process, found no
  value drift and no failed read outside sections 2.2 and 2.5 (OBSERVED).
- No standby or pixel-cleaning period was deliberately observed. Whether DDC/CI answers while the proximity sensor has
  blanked the screen, in standby or during a cleaning is UNKNOWN on this unit; on a PG32UCDM the monitor drops off
  DDC/CI during cleaning (**PUBLISHED**, pixel-clean-gui).

## 7. Writing: guidance (nothing here was sent to this unit)

No `SetVCPFeature` was ever sent to this monitor (OBSERVED, section 2.1). The rules below combine the published
practice of sumi (read-modify-write of packed codes, **PUBLISHED**), KDE PowerDevil (read before write, **PUBLISHED**,
merge request 456) and the readings above; they are INFERRED as safe, not tested.

1. Read the code fresh. Refuse when it reads `0xFE`/`0xFE` (section 6.1) or when the read fails.
2. Refuse a value outside the capability string's list for that code (or above the read maximum for a plain number).
3. Write nothing if the field already has the value: each write may be stored in the monitor's non-volatile memory, and
   repeated NVRAM writes are reported to wear some monitors (**PUBLISHED**, KDE PowerDevil merge request 456; INFERRED
   for this model).
4. Build the word from the fresh read, changing only the target byte or bit (`0xDC`, `0xED`, `0xF2`, `0xFB`, `0xFC`,
   `0xFD`). Never build a word from defaults or from an older read, and never use arithmetic such as "current + 16" to set
   a bit (pixel-clean-gui does; it corrupts the word when the bit is already set, INFERRED).
5. For `0xFD`, clear `0x0010` in the word you write unless you mean to start a pixel cleaning, and refuse every `0xFD`
   write while `0x0010` reads 1 (**PUBLISHED**, sumi's rule; INFERRED for this unit).
6. Send `SetVCPFeature`, wait at least 50 ms, read the code back, and re-read its dependants: the four per-mode codes
   after `0xDC` (section 6.2), `0x14` and `0x1A` after `0xE6` (section 6.3), `0xE5`, `0xE6`, `0xF4` and `0x10` after
   anything that can lock them.
7. Never call `SaveCurrentSettings` (**PUBLISHED**, Microsoft: it writes the monitor's non-volatile storage).
8. One transaction at a time per monitor, and expect another program (DisplayWidget Center, PowerToys, Twinkle Tray)
   to write the same codes: read back once and keep the other program's value rather than re-applying in a loop
   (INFERRED; no system-wide DDC/CI lock is documented).

What the monitor does with a write to a locked control (an error, silent ignore, or a later application) is UNKNOWN.

## 8. Sequences

### 8.1 Read-only identification

1. In the user's session, get the physical-monitor handle (section 2.1) and match it to the EDID (section 1.3).
2. Read `0xEF` and `0xFF`. A maximum of `0xEF` with high byte 2 means the map of sections 4 and 5 applies (PUBLISHED
   rule, OBSERVED on this unit); on anything else use the standard codes only.
3. Use a cached capability string; read it live only on an explicit request (section 3.1).
4. Read the listed codes, current and maximum; skip `0x04`, `0x08` (write-only resets) and expect `0xE8`, `0xEB`,
   `0xEC` to fail with `0xC0262584`.

### 8.2 The OSD-correlation method used here

Read every readable listed code (52 here), change **one or two** OSD items by hand, close the OSD, read again and diff;
repeat. A first pair of reads with no change shows which codes drift by themselves (only `0xC0` here). Change items
whose codes might interact (for example Uniform Brightness and brightness) in separate rounds, and finish by putting
everything back and checking the final read against the first (OBSERVED, the method of 2026-11-01). The method sends only
"Get VCP Feature"; sumi ships a read-only diff command that does the same (**PUBLISHED**, sumi README).

## 9. USB path: the Aura HID (from published sources and Windows properties only)

Nothing was sent to any USB function of this monitor, and no handle was opened (HOST: the third-party code has no driver
for these devices; OBSERVED by absence in its logs). This section combines Windows' records with what open-source
projects published for sibling monitors.

### 9.1 USB functions behind the monitor's hub (HOST, 2026-10-21)

| Function | IDs | Windows binding |
|---|---|---|
| internal hub, USB 2 side | `0B05:1C1F`, bcdDevice `0133`, bus description "USB2.1 Hub" | inbox hub driver |
| internal hub, SuperSpeed side | `0B05:1C20`, "USB3.2 Hub" | not present at read time (seen earlier) |
| Aura device, composite | `0B05:1C2B`, bcdDevice `0100`, bus description "ROG Gaming Display Aura Device", on hub port 4 | `usbccgp`; Microsoft OS descriptor vendor code `0x87` |
| Aura interface 0 | class `FF/FF/FF`, compatible id `USB\MS_COMP_WINUSB` | WinUSB, device-interface GUID `{6987B675-8CBC-4BD3-A557-C627CEA223DC}` |
| Aura interface 1 | HID, one top-level collection, hardware id `HID\VID_0B05&UP:FF72_U:00A1` (usage page `0xFF72`, usage `0xA1`) | inbox `HidUsb`; selective suspend disabled |
| Realtek HID | `0BDA:1100`, bcdDevice `0101`, "HID Device", on hub port 6; usage page `0xFFDA`, usage `0xDA` | inbox `HidUsb`; selectively suspended |

- The Aura device reports a placeholder-style USB serial number, not the panel serial (not reproduced here), and its
  USB container id differs from the display's, so Windows does not link the Aura device to the display (HOST). Pair
  them by "one ASUS monitor present and its Aura device behind hub `0B05:1C1F`" (INFERRED).
- The WinUSB GUID and vendor code are identical to those of the Lian Li UNI HUB SL V2's interface 0, an ENE-based
  controller on the same PC, and the usage pair `0xFF72`/`0xA1` on interface 1 is also that hub's (HOST). So the pair
  marks an ENE USB firmware framework, not ASUS's command set (INFERRED).

### 9.2 The Aura protocol published for sibling monitors (**PUBLISHED**; UNKNOWN on this unit)

OpenRGB's `AsusMonitorController` (`Controllers/AsusMonitorController/`) drives the PG32UCDM, PG32UCDP, XG27AQDMG,
XG27ACDNG, XG27UCG, PG32UCDMR and the ROG Aura Monitor Light Bar over interface 1 with usage page `0xFF72`/usage `0xA1`
(**PUBLISHED**). `0x1C2B` is in no OpenRGB detector (**PUBLISHED**, by absence, as of October 2026). Its packets are
65-byte reports with report ID `0xEC`:

| Packet | Bytes from `[0]` (rest `00`) | Answer | Purpose |
|---|---|---|---|
| configuration query | `EC B0` | input report, `[1]` = `0x30` on the mainboard family; LED count at `[32]` on monitors | read |
| direct-mode init | `EC 35 00 00 00 FF 00 00 01` | none | effect channel 0 to direct mode (`FF`); the meaning of `[8]` = `01` is UNKNOWN |
| direct frame | `EC 40 84 00 nn` + `nn` × R, G, B | none | colours shown at once; `0x84` = direct channel 4 with the apply flag `0x80` |

- At most 20 LEDs fit one 65-byte frame (5 + 3 × 20); the mainboard family splits larger counts with an offset in `[3]`
  (**PUBLISHED**, OpenRGB `AsusAuraUSBController` and hypercolor `crates/hypercolor-hal/src/drivers/asus/`). Siblings
  report 3 or 5 LEDs (**PUBLISHED**, OpenRGB merge request !3539 and issue #5690 for the PG34WCDN). This model has lit
  areas on the rear cover and the base cover (**PUBLISHED**, ASUS user guide p. 3-11); its LED count is UNKNOWN.
- OpenRGB sends `EC B0` and the `EC 35` init as soon as it detects a device (**PUBLISHED**): a build that matched
  `0x1C2B` would switch the Aura controller to direct mode at every start (INFERRED).
- The mainboard family also has a firmware query `EC 82`, a commit `EC 3F 55` used only by OpenRGB's "save to device"
  path, and a mode switch `EC 52 53 00 01` (**PUBLISHED**, OpenRGB). No monitor source sends them. Do not send them to a
  monitor (INFERRED: the commit is a flash write on mainboards).
- ASUS's user guide asks for OSD Aura Sync on, a USB-C connection to the PC, Armoury Crate, and a PC restart after the
  USB-C cable is reconnected (**PUBLISHED**, p. 3-16). Whether direct frames are ignored with Aura Sync off is UNKNOWN.
- Whether `0x1C2B` speaks this protocol at all is UNKNOWN. Evidence for: same interface number and usage as the
  siblings, and the PG34WCDN, another ROG OLED, works with it unchanged (**PUBLISHED**, OpenRGB issue #5690, a user
  report). Evidence that proves nothing: the product string, which old-protocol ASUS monitors share (**PUBLISHED**,
  OpenRGB issues #1689, #3137, #3859, #4139). A decisive read-only check is the report ID in the HID report descriptor
  (`0xEC` expected; `0xE0` would mean a Lian Li-style ENE command set), then one `EC B0` (INFERRED).
- For effects that the monitor stores, `0xF2` over DDC/CI (section 5.7) needs no USB at all (OBSERVED for the read side).

### 9.3 Interface 0 and the Realtek HID: do not use

- On the ROG Aura Monitor Light Bar the endpoint-less vendor interface 0 serves vendor requests `C0 81` (read) and `40 80`
  (write) on 64-byte blocks, and a request with a wrong `wValue` stalls the controller until a USB reset (**PUBLISHED**,
  OpenRGB merge request !3548, open). On the ENE hub that shares this interface's GUID, interface 0 is the firmware
  updater's channel (**PUBLISHED**, the Lian Li UNI FAN SL V2 reference). Interface 0 of `0x1C2B` is most likely the
  ENE controller's register and firmware channel (INFERRED). Never open it; Windows binds WinUSB to it automatically,
  so it opens without elevation (HOST, INFERRED).
- `0BDA:1100` with the same bcdDevice appears as "Realtek USB2.0 HID" on the last port of Realtek hubs inside other
  monitors and docks (**PUBLISHED**, linux-hardware.org and user reports). It is most likely the hub controller's own
  vendor HID (INFERRED, low). Never open it.
- ASUS updates the monitor's firmware with a separate tool (**PUBLISHED**, ASUS). Do not trigger it.

## 10. ASUS software

### 10.1 What the monitor's driver package installs (HOST and METADATA, 2026-10-21)

DisplayWidget Center itself was never installed on the observed PC (HOST). The monitor's Windows driver package carries
only a bootstrapper for it:

| File | What it is (strings, version resources and .NET metadata only; nothing run) |
|---|---|
| `DisplayWidgetCenterInstallService.exe` | a .NET Framework 4.7.2 service, internal name `AsusDisplayAppInstallService.exe`, version 1.0.0.3, signed "ASUSTeK COMPUTER INC., Display BU". Registered by the INF as service `DisplayWidgetCenterInstallService`, demand start, LocalSystem, with the add-service flag `0x800` (start the service at install), so **Windows starts it once whenever the driver package is installed** (HOST: INF and registry). It launches the helper below in the active console session (`WTSGetActiveConsoleSessionId`, `WTSQueryUserToken`, `CreateProcessAsUser` on `winsta0\default`) (METADATA) |
| `DisplayWidgetCenterInstallHelper.exe` | .NET Framework 4.7.2, internal name `AppInstallHelper.exe`, version 1.0.0.5, same signer. Looks for "DisplayWidget Center" among the installed programs; if missing, downloads `DisplayWidgetCenterDownloader.exe` from ASUS's download server, checks its signature and runs it (METADATA) |

- Neither file contains a monitor protocol: no `dxva2`, HID, WinUSB or SetupAPI imports or strings (METADATA).
- An empty temporary folder named `DisplayWidgetCenterInstaller`, dated 2026-09-04, sat in the user's temp directory
  (HOST): the helper most likely ran once at driver install without completing an installation (INFERRED).
- Consequence: reinstalling, disabling and re-enabling, or updating the monitor's driver can start the download of
  ASUS's software again (INFERRED from the service flag). A machine meant to run without ASUS software should leave
  the monitor's driver alone, or use Windows' generic monitor driver.

### 10.2 How DisplayWidget Center and ASUS's CLI reach the monitor

- ASUS publishes "ASUS Display Control CLI" (`dwc.exe`, binaries and documentation only), released alongside a
  DisplayWidget Center update. It controls ASUS monitors over DDC/CI, needs DDC/CI on in the OSD and a DisplayPort or
  HDMI connection, and has no USB requirement; it offers raw `getcaps`, `getvcp` and `setvcp` and tells users to read
  the capabilities of the actual monitor first (**PUBLISHED**, ASUS-Display/asus-display-control,
  `cli/docs/CLI_REFERENCE.md` and `skills/asus-display-control/SKILL.md`; Hardware Busters news article).
- Its property values match this unit's raw values wherever both are known: GameVisual 1 Cinema to 10 sRGB Calibration,
  colour temperature 3-9 and 11, input 15/17/18/26, OSD language up to 39, EZOSD keys 0-7, `AuraMode` 0-6, `AuraColor`
  0-6 (**PUBLISHED**, CLI reference; OBSERVED values here). So on this scheme the CLI's numbers are the raw VCP values
  (INFERRED). The CLI does not name the codes.
- The CLI has OLED-care properties for pixel cleaning (`OledPixelClean`, 0 stop, 1 start), Screen Move, the reminder,
  uniform brightness, the dimming and detection toggles and Anti-Flicker (**PUBLISHED**). The proximity sensor is not
  among them, although this unit exposes it in `0xED` (OBSERVED).
- DisplayWidget Center's App Tweaker and AI Visual change the GameVisual mode on their own (**PUBLISHED**, ASUS FAQ
  1050223 and product page); sumi's reimplementation of AI Visual writes only `0xDC` (**PUBLISHED**, sumi README).
  INFERRED: DisplayWidget Center's settings path is DDC/CI; whether it also uses a USB function of this model is UNKNOWN
  (no capture). On the ROG Aura Monitor Light Bar a lamp that DisplayWidget Center controls sits on the USB vendor
  interface (**PUBLISHED**, OpenRGB merge request !3548), so "DDC/CI only" is not a rule for every ASUS product.
- Armoury Crate (Aura over USB) and DisplayWidget Center were not installed during any observation (HOST). An OpenRGB
  build without a detector for `0x1C2B` was installed and running during the probe (HOST).

## 11. Hazards

| Hazard | Guard |
|---|---|
| A service cannot reach the monitor (session 0) | Run DDC/CI in the user's session (section 2.2); report "no user session" instead of retrying in a loop |
| Capability request crashing Windows on some monitors (**PUBLISHED**, PowerToys) | Read the string once on explicit request, cache it; this unit's string is in section 3.1 |
| Probing codes the string does not list can black the screen, change settings or stop DDC/CI until a power cycle (**PUBLISHED**, PowerToys) | Read only listed codes; never scan `0x00`-`0xFF` |
| `0x61`: writing it restarts the monitor (**PUBLISHED**, plasma-asus) | Never write `0x61` |
| Write-only commands: `0x04`, `0x08` (resets), `0xEC` (ASUS reset), `0xEB` (OSD key injection), `0xE8` (GamePlus nudge) | Never send them automatically; resets only as an explicit user action |
| Starting or stopping pixel cleaning by accident: a whole-word or stale write of `0xFD` with `0x0010` set; the CLI's "stop" value | Read-modify-write of one bit, `0x0010` cleared; refuse while it reads 1 (section 7) |
| Weakening burn-in protection (Screen Move, the dimming and detection bits, the reminder, the proximity sensor) on a QD-OLED panel | Only as a direct user action, never from schedules or profiles (INFERRED) |
| Waking the monitor during its automatic pixel cleaning in standby stops the cleaning (**PUBLISHED**, ASUS user guide p. 3-8) | Never send `0xD6` = on automatically; leave a sleeping monitor alone |
| Two settings in one code (`0xDC`, `0xED`, `0xF2`, `0xFB`) or a bit field (`0xFC`, `0xFD`) | Change one byte or bit of a fresh read; Aura off as `0x0000` (**PUBLISHED**, plasma-asus) |
| Writes to locked controls | Refuse at `0xFE`/`0xFE`; in HDR and under Uniform Brightness or Anti-Flicker, brightness reads 100 instead of the marker (section 6.1) |
| HDR family (`0xE2` high byte) and PBP (`0xF4`) changes re-plug the display (**PUBLISHED**, sumi) | Explicit actions only; expect handle invalidation (`0xC026258D`) |
| Input switch (`0x60`) or power off (`0xD6`) leaving a black screen and no DDC/CI (**PUBLISHED**, PowerToys) | Explicit actions with confirmation, never automated |
| KVM (`0xF3`) moving the USB hub, the Aura device and the user's keyboard to another host (INFERRED) | Never automatic |
| NVRAM wear from repeated writes (**PUBLISHED** concern, KDE PowerDevil; INFERRED here) | Read-compare-write, debounce sliders, rate-limit automation; never `SaveCurrentSettings` |
| Blue Light Filter leaving the colour preset at User 1 | Record `0x14` first (section 6.3) |
| Applying another ASUS model's map (on older models `0xE9`/`0xEB` lock keys) | Check `0xEF`'s maximum before using any ASUS code (section 1.4) |
| Opening USB interface 0 of `0x1C2B` or `0BDA:1100` (register, firmware, stall) | Never open them; accept only interface 1 with usage `0xFF72`/`0xA1` for Aura |
| Aura USB init switching the controller to direct mode (**PUBLISHED**, OpenRGB) | Send nothing on interface 1 until the protocol is confirmed (section 9.2) |
| Reinstalling the monitor driver restarts ASUS's download bootstrapper (section 10.1) | Leave the driver alone |
| Firmware update | **Do not trigger.** |

## 12. Test vectors

**OBSERVED readings** (current of maximum, this unit, 2026-11-01):

| Id | Reading | Decoded |
|---|---|---|
| T1 | `0xEF` `0x00B1` of `0x0211` | scheme 2.17 (major 2 = new layout) |
| T2 | `0xFF` `0x0030` of `0x0065` | firmware 101, panel id `0x30` |
| T3 | `0xFD` `0x7829` of `0x787D` | VRR, Screen Dimming, Logo, Taskbar, Boundary, Outer, Global on; Sniper, cleaning, Uniform off |
| T4 | `0xFC` `0x0811` of `0x089F` | power indicator on, Auto Input Detection on, `0x0800` unnamed |
| T5 | `0xDC` `0x0205` of `0x020A` | GameVisual Racing, colour space DCI-P3 |
| T6 | `0xE2` `0x0102` of `0x0104` | HDR, Gaming |
| T7 | `0xED` `0x0A01` of `0xFFFF` | screen off after 10 min, 120 cm |
| T8 | `0xF2` `0x0304` of `0x0606` | Static, blue |
| T9 | `0x72` `0x6400` of `0xA000` | gamma 2.0 |
| T10 | `0xE6` `0x00FE` of `0x00FE` | locked |
| T11 | `0xC0` `0x0448` of `0x0000` | 1096 hours |
| T12 | Get `0xEB` | fails, `0xC0262584` |

**OSD transitions, as a bit-or-byte encoder must reproduce them** (OBSERVED words before and after):

| Id | Change | Before | After |
|---|---|---|---|
| T13 | Uniform Brightness on (`0xFD` set `0x0040`) | `0x7829` | `0x7869` |
| T14 | VRR off (`0xFD` clear `0x0001`) | `0x7829` | `0x7828` |
| T15 | Power indicator off (`0xFC` clear `0x0001`) | `0x0811` | `0x0810` |
| T16 | Display Color Space sRGB (`0xDC` high byte `01`) | `0x0205` | `0x0105` |
| T17 | GameVisual FPS (`0xDC` low byte `07`) | `0x0001` | `0x0007` |
| T18 | proximity 60 cm (`0xED` low byte `03`) | `0x0502` | `0x0503` |
| T19 | Aura Static blue (`0xF2` high byte `03`) | `0x0104` | `0x0304` |

**Built from the layouts** (INFERRED arithmetic; never sent):

| Id | Case | Expected |
|---|---|---|
| T20 | capability tokens `DC(0000 0100 0200 01 02 04 05 06 07 08 09 0A)` | high-byte values {0, 1, 2}; low-byte values {1, 2, 4, 5, 6, 7, 8, 9, 10} |
| T21 | `FD(787D)` | support mask `0x787D`, equal to `0xFD`'s maximum |
| T22 | `72(50 64 78 8C A0)` | gamma 1.8, 2.0, 2.2, 2.4, 2.6 in the high byte |
| T23 | `mccs_ver(2.17)` | ASUS scheme 2.17, not an MCCS version |
| T24 | Logo Detection off from `0x7829` | `0x7809`; refused while Global Dimming (`0x4000`) is on, since the monitor locks Logo on |
| T25 | any `0xFD` change when the read is `0x7839` | refused: `0x0010` set (cleaning running, sibling rule) |
| T26 | Aura off from `0x0304` | write `0x0000`, not `0x0300` |
| T27 | set proximity 90 cm from `0x0F00` | `0x0F02` |

**Anti-vectors**: a parser that reads `0100` in `DC(...)` as one byte; reading `0x72`'s value from the low byte; a
brightness level of 254 from a locked control; using `0xE2` = `0x0104` alone to claim HDR is on; a whole-word `0xFD` write
built from defaults; treating 2.17 as an MCCS version.

## 13. Open items

| # | Question | How to settle |
|---|---|---|
| 1 | Does the monitor refuse, ignore or store a write to a locked control (HDR, Uniform Brightness, Anti-Flicker)? | one same-value write in HDR, then a read and the OSD |
| 2 | Is `0xFD` bit `0x0010` the pixel-cleaning trigger here, and does DDC/CI answer during a cleaning, in standby or after a proximity screen-off? | the owner starts a cleaning from the OSD and reads `0xFD` once afterwards; a read while the sensor has blanked the screen |
| 3 | Meanings of `0xE9`, `0xF0`, `0xF7`, `0xFA`, `0x5F`, `0x61`, and the `0xFC` bits `0x0002`, `0x0004`, `0x0008`, `0x0080`, `0x0800` | more OSD rounds (OSD timeout and transparency, HDR modes, USB settings) |
| 4 | `0xE2` with each OSD HDR mode; Adjustable HDR and Dynamic Brightness Boost; the high byte | OSD rounds with Windows HDR on |
| 5 | GamePlus codes `0xE3`, `0xE4`, `0xE7`, `0xEA`, the `0xE8` values `05`-`08`, and `0xFB`'s function ids | OSD rounds |
| 6 | KVM `0xF3`: which value is which upstream, and does a write move the hub? | an OSD round; a write only with a second host attached |
| 7 | `0xED` low byte `00` and `FF` (off, Tailored); can presence be read? | OSD rounds; reads while walking away |
| 8 | Does `0xF2` drive the LEDs with Aura Sync on and no USB host? | watch the LEDs with Aura Sync on and the USB cable out |
| 9 | Does `0x1C2B` speak the `0xEC` protocol, how many LEDs, and does Aura Sync off block USB frames? | the HID report descriptor (read-only), then one `EC B0` |
| 10 | Whether DisplayWidget Center uses any USB function of this monitor | a USB capture of DisplayWidget Center changing one setting |
| 11 | What interface 0 of `0x1C2B` and `0BDA:1100` are | published captures of other ASUS monitors; never probe them |
| 12 | Why `0x62` moved 51 → 50 at the HDR snapshot | another HDR on/off round with no other change |
| 13 | Whether `0x5F(0005)` is a list, range or mask | more ASUS capability strings |

Firmware update paths are deliberately not on this list.
