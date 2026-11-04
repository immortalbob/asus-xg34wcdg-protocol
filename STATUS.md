# Status and change log

Part of the [ASUS ROG Strix OLED XG34WCDG protocol reference](README.md). One row per code or area; the tags are
explained in the README. "Read" means the code answered the probe of 2026-11-01 and every later read; "OSD-correlated"
means the owner changed the named OSD item and only this code moved accordingly (2026-11-01). **No row was ever
written to the monitor.**

## DDC/CI

| Code / area | Meaning | Encoding | Read | OSD-correlated | Open |
|---|---|---|---|---|---|
| API and session | dxva2, user session only; session 0 sees no monitor | complete | yes | n/a | |
| capability string | 57 codes, `mccs_ver(2.17)` | complete; token forms decoded | yes (once) | n/a | `5F(0005)` |
| `0xEF` / `0xFF` | scheme 2.17 / firmware 101, panel id | complete (PUBLISHED rule) | yes | n/a | meaning of the current values |
| NC maximum field | passed through by dxva2 | complete | yes | n/a | |
| `0x02`, `0x52` | MCCS change flags | complete | yes; never move | no change in 38 snapshots | |
| `0x04`, `0x08` | MCCS resets (write-only) | complete (PUBLISHED) | not read | | **do not send** |
| `0x10` | brightness | complete | yes | per-mode values seen | behaviour of a write under a lock |
| `0x12` | contrast | complete | yes | no | |
| `0x14` | colour preset | complete | yes | 6500 K, 7500 K, User 1 | |
| `0x16` / `0x18` / `0x1A` | RGB gains | complete | yes | `0x1A` with the Blue Light Filter | |
| `0x59`-`0x5E` | six-axis saturation | MCCS codes, 0-100 scale | yes | no | |
| `0x5F` | UNKNOWN (Adjustable HDR?) | | yes | moved with HDR | meaning; the `0005` token |
| `0x60` | input | complete | yes | no | |
| `0x61` | UNKNOWN | | yes | no | **do not write** (restarts the monitor, PUBLISHED) |
| `0x62` | volume | complete | yes | no | the step at the HDR snapshot |
| `0x72` | gamma, high byte | complete | yes | 2.2 and 2.0 (per mode) | |
| `0x87` | VividPixel | complete | yes | 50 → 60 | |
| `0x8A`, `0x8D` | saturation, mute | complete | yes | no | |
| `0xC0` | usage hours | complete | yes; counts up | n/a | |
| `0xCC` | OSD language | complete | yes | no | |
| `0xD6` | power mode | complete (PUBLISHED) | yes | no | wake after "display off" |
| `0xDC` | GameVisual (low) and colour space (high) | complete | yes | 3 modes, 3 colour spaces | six modes unseen |
| `0xE1` | Power Setting | PUBLISHED (sibling) | yes | no | |
| `0xE2` | HDR-mode word | low byte complete; high byte 0 or 1 | yes | Anti-Flicker; Windows HDR (2 values) | each OSD HDR mode; Adjustable HDR |
| `0xE3`, `0xE4`, `0xE7`, `0xEA` | GamePlus crosshair, timer, alignment, FPS | PUBLISHED (sibling) | yes | no | all |
| `0xE5` | Shadow Boost | complete | yes | off → 1; per mode | |
| `0xE6` | Blue Light Filter | complete | yes | 0 → 1 → 0 with side effects | |
| `0xE8`, `0xEB`, `0xEC` | write-only commands | PUBLISHED (sibling) | fail, `0xC0262584` | n/a | **do not send** |
| `0xE9` | UNKNOWN | | yes | no | meaning |
| `0xED` | Neo Proximity Sensor | complete for 1-3 and 5/10 min | yes | distance and timer | `00`, `FF`; presence readout |
| `0xEE` | ELMB? (PUBLISHED) | | lock marker always | n/a (greyed out) | at 87 Hz with VRR off |
| `0xF0`, `0xF7`, `0xFA` | UNKNOWN | | yes | `0xFA` moved with HDR | meaning |
| `0xF2` | Aura effect (low) and colour (high) | complete | yes | 4 effects, 2 colours | Aura Sync without a USB host |
| `0xF3` | KVM | PUBLISHED (sibling) | yes | no | which value is which upstream |
| `0xF4`-`0xF6` | PIP/PBP | PUBLISHED (sibling) | yes | lock marker seen | all |
| `0xF8` | cleaning reminder | complete | yes | 8 → 4 → 8 | |
| `0xF9` | Screen Move | complete | yes | middle → light → middle | |
| `0xFB` | shortcut keys | layout PUBLISHED | yes | no | function ids |
| `0xFC` | bit field | 2 of 7 bits named | yes | power indicator, Auto Input | 5 bits |
| `0xFD` | bit field | all 10 bits named | yes | 9 bits | the cleaning bit `0x0010` on this model |
| lock marker | `0xFE` of `0xFE` | complete | yes | 4 codes, 3 conditions | write behaviour |
| per-mode storage | brightness, preset, gamma, Shadow Boost | complete for 3 modes | yes | yes | |
| stability | 60 s polling for 2.5 days, three sleeps | | yes | n/a | standby, cleaning, proximity screen-off |
| writes | | guidance only | **never sent** | | everything about writing |

## USB

| Function | Status | Open |
|---|---|---|
| hub `0B05:1C1F` / `0B05:1C20` | HOST only | |
| Aura `0B05:1C2B` interface 1 (HID, `0xFF72`/`0xA1`) | HOST; protocol PUBLISHED for siblings only; nothing sent | report ID, LED count, whether `EC B0` answers |
| Aura `0B05:1C2B` interface 0 (WinUSB) | HOST; **do not use** | its purpose (INFERRED: ENE register and firmware channel) |
| Realtek `0BDA:1100` HID | HOST; **do not use** | its purpose |
| firmware update | **deliberately not described** | |

## ASUS software

| Item | Status |
|---|---|
| DisplayWidget Center bootstrapper in the driver package | METADATA and HOST complete |
| DisplayWidget Center | never installed; its USB use UNKNOWN |
| ASUS Display Control CLI | PUBLISHED documentation only; its values match the raw values seen |
| Armoury Crate | never installed |

## Change log

- **2026-11-04, first edition.** Written from the read-only DDC/CI probe of 2026-11-01 (capability string and every
  listed code), 38 OSD-correlation snapshots of the same evening (35 change rounds and one reading with Windows HDR on),
  a session-0 comparison, read-only polling from 2026-11-01 to 11-04, Windows' device properties, registry and EDID,
  the strings and metadata of the two ASUS executables in the monitor's driver package, and the open-source projects
  and documents credited in the README. New compared with what was public before: the XG34WCDG's capability string and
  scheme 2.17; the sibling map confirmed code by code; Display Color Space in `0xDC`, the proximity sensor in `0xED`,
  Aura in `0xF2`, the power indicator in `0xFC`, Sniper in `0xFD`, VividPixel on `0x87`; `0xE2` as the HDR word with
  Anti-Flicker's `0x0104`; the lock marker; per-mode storage; the Blue Light Filter's side effect; dxva2's handling of
  maximum fields, session 0 and handle invalidation; DisplayWidget Center's bootstrapper.
