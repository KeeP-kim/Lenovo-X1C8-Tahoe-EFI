# Lenovo-X1C8-Tahoe-EFI

The Last Intel Custom Mac — OpenCore EFI for the Lenovo ThinkPad X1 Carbon Gen 8.

**Current build: macOS Tahoe 26.6.2 (25G83), OpenCore 1.0.7**

Sanitized for public use. Personal SMBIOS values are placeholders and **must** be replaced
with your own before booting.

## Configuration

This EFI is maintained on a **rebodied** machine: the SSD was moved into a different X1 Carbon
Gen 8 chassis on 2026-08-31. Everything below reflects the current hardware.

| | |
|---|---|
| Model | Lenovo ThinkPad X1 Carbon Gen 8 (20UAS8NE00) |
| CPU | **Intel i7-10610U** (4C/8T, Comet Lake-U) — was i5-10210U |
| RAM | **16 GB** — was 8 GB |
| Panel | **BOE**, 1920x1080 (`DisplayVendorID 0x9e5` / `ProductID 0x7db`) — was AUO `0x6af` |
| BIOS | N2WET52W (1.42) |
| macOS | Tahoe 26.6.2 (25G83) |
| OpenCore | 1.0.7 |
| SMBIOS | MacBookPro16,2 |
| Audio | Realtek ALC285 via VoodooHDA (`voodoo-layout-id=71`) |
| Wi-Fi | Intel AX201 via itlwm |
| Trackpad | VoodooI2C + VoodooI2CHID |
| Keyboard | VoodooPS2Controller |

No `config.plist` changes were needed for the CPU or RAM change — the hardware IDs all matched.
**The panel swap did break HiDPI**, because display overrides are keyed by panel ID; see the
HiDPI section below.

### What works

Boot from the internal disk with no USB, audio out, internal mic, volume control, HiDPI,
keyboard media keys, all four USB ports at 5 Gbps, Wi-Fi, Bluetooth, trackpad, battery, and
sleep/wake.

`hibernatemode` is **0** — RAM stays powered and nothing is written to disk, so there is no
hibernation / "deep sleep" on this setup. That is the normal choice for a hackintosh; do not
raise it expecting it to work.

Windows boots from the firmware boot menu (F12), not from the OpenCore picker. See the note at
the end.

## Before you boot: fill these in

`EFI/OC/config.plist` → `PlatformInfo > Generic`

```
SystemSerialNumber = CHANGEME_SERIAL
MLB                = CHANGEME_MLB
SystemUUID         = 00000000-0000-0000-0000-000000000000
ROM                = 000000000000
```

Generate your own with GenSMBIOS. Never reuse someone else's serials.

---

## Three settings that are easy to get wrong

These cost a lot of debugging to find. If you copy nothing else, copy these.

### 1. `Misc > Security > SecureBootModel = Disabled`

With `Default`, this SMBIOS resolves to `j214kap` and **every OTA update fails**. The picker's
"macOS Installer" entry just loops forever. Confirm the cause by reading `ota-failure-reason`
in `/System/Volumes/Update/last_update_result.plist` — it will say:

```
MSU 1130 (Failed to verify against canonical metadata);
NSPOSIXErrorDomain 13 (img4_firmware_execute failed)
```

### 2. `DeviceProperties` → `PciRoot(0x0)/Pci(0x1F,0x3)` → `voodoo-layout-id = 71` (`47000000`)

macOS 26 removed `AppleHDA.kext` entirely, so **AppleALC never loads** and VoodooHDA is the only
analog audio path. This VoodooHDA fork embeds AppleALC's pin table and reads
`voodoo-layout-id` > `layout-id` > plist `LayoutId`.

Most ALC285 layouts leave `nid 0x17` unset, so it falls back to its BIOS default `0x90170111`
(Speaker, **as=1**) — colliding with the internal mic at `nid 0x12` (**as=1**). VoodooHDA then
disables the entire association:

```
Pin 23 has wrong direction for association 1! Disabling association.
Association 0 (1) in (disabled)
```

and you lose all audio input.

**Layout 71 is the only ALC285 layout that explicitly moves `nid 0x17` to as=4**, leaving the mic
(as=1) and speaker (as=3) in clean, separate associations. Result: 2-channel output, working
microphone, working volume control.

> Layout 99 (a deliberate table miss, so every pin falls back to BIOS defaults) also revives the
> mic — but groups output into a bogus 4-channel rear-jack association. The volume slider moves
> and nothing gets louder.

**To verify this yourself without rebooting:** download an AppleALC release and decode
`AppleALC.kext/Contents/Info.plist` → `IOKitPersonalities` → `as.vit9696.AppleALC` →
`HDAConfigDefault`. Filter `CodecID == 0x10EC0285` and parse each `ConfigData` as 32-bit HDA
verbs — NID is bits 27–20, verb is bits 19–8, pin-config verbs are `0x71C`–`0x71F`, and the
payloads assemble as bytes 3..0. In the resulting pin config, bits 7–4 are the association and
bits 3–0 the sequence.

### 3. `UEFI > Quirks > RequestBootVarRouting = True` — and when to turn it off

`True` is correct for normal use, but it **silently blocks creating a firmware boot entry**.
It redirects `Boot####` / `BootOrder` access from the EFI global GUID to OpenCore's private
vendor GUID, so `bless --setBoot` returns exit 0, `bless --info --getBoot` reports the right
disk, and the firmware never learns anything.

If your NVRAM boot entries are gone (e.g. after a mainboard swap) and the machine boots straight
to Windows:

1. Set `RequestBootVarRouting = False` in the config of the OpenCore you are actually running
   (a USB stick, if that is what boots)
2. Reboot, then:
   ```
   sudo bless --mount <ESP> --setBoot --file <ESP>/EFI/OC/OpenCore.efi
   ```
3. Verify — note that `nvram -p` does **not** list global-GUID variables:
   ```
   nvram -x "8BE4DF61-93CA-11D2-AA0D-00E098032B8C:BootOrder"
   ```
4. Restore `RequestBootVarRouting = True`

Also check for a malformed leftover entry. A `Boot####` pointing at `\OC\OpenCore.efi` (missing
the `\EFI` prefix) will fail and let the firmware fall through to Windows.

### 4. Laptop EC: do **not** use the desktop `SSDT-EC` pattern

If your battery percentage never appears, check this before anything else.

A desktop-oriented ACPI setup renames the real Embedded Controller away and substitutes a stub:

```
SSDT-EC.aml               Enabled = true    creates a fake EC, _HID ACID0001
ACPI > Patch  EC to EC0                     renames the real EC
ACPI > Patch  EC _STA to XSTA               disables the real EC
```

On a laptop the battery (`BAT0`) lives **under the real EC**, so this removes it from the ACPI
tree entirely. `SMCBatteryManager` loads fine and then has nothing to attach to — `pmset -g batt`
reports only `AC Power` with no battery line, and there is no `AppleSmartBattery` node in ioreg.

On a ThinkPad it also silently breaks `SSDT-THINK.aml`, which references
`_SB.PCI0.LPCB.EC.HKEY` / `.HFSP` / `.HFNI` — methods that exist only on the real EC.

**Fix: disable all three.** Keep `SSDT-USBX.aml` (it defines `\_SB.USBX` and is independent).

Verify afterwards — the EC node's `name` should be the real `PNP0C09`, not `ACID0001`:

```
ioreg -w0 -p IOACPIPlane -n EC -r | grep name
pmset -g batt
```

---

## Keyboard notes (VoodooPS2)

The `fn` key emits **no HID event at all** on this machine, so `fn` combinations cannot be
distinguished in software. FnLock is the only usable discriminator.

If `Custom PS2 Map` in `VoodooPS2Keyboard.kext` contains entries rewriting the stock media
scancodes to F-keys, F1–F3 will behave identically in both FnLock states. For
FnLock ON = media / FnLock OFF = real F-keys, use a bidirectional swap:

```
3b=e020   e020=3b     F1 <-> Mute
3c=e02e   e02e=3c     F2 <-> Volume Down
3d=e030   e030=3d     F3 <-> Volume Up
```

## HiDPI

HiDPI overrides are matched by panel `DisplayVendorID` / `DisplayProductID`, which differ per
individual panel — a mainboard or panel swap breaks them. Read the current IDs from `ioreg` and
recreate `/Library/Displays/Contents/Resources/Overrides/DisplayVendorID-<hex>/DisplayProductID-<hex>`.

**Keep the downscale ratio at or below 1.5x.** A 1080p panel driven from a 3360x1890 backing
store is a 1.75x downscale, which sits at the practical limit of the Gen9.5 pipe scaler: it looks
fine on a cold boot but on resume from sleep the scaler window is not re-armed and you get a small
image anchored in one corner with the rest of the screen black. Dropping the largest mode so the
worst ratio is ~1.5x fixed it here, verified over 50 sleep/wake cycles.

This is **not** a framebuffer memory problem, and raising `framebuffer-stolenmem` /
`framebuffer-fbmem` to "make room" is actively dangerous — those are additive and this Lenovo
firmware locks DVMT pre-allocation at 32 MB, so overshooting black-screens the machine at boot.
The backing store is GART-mapped system memory: `inUseVidMemoryBytes` reads 0 while a 2848x1604
mode is active.

## Known limitations

- Putting kexts in `/Library/Extensions` requires a KDK matching the exact build; every minor
  macOS update invalidates the AuxKC. Prefer OpenCore injection.
- `XhciPortLimit = true` is used and `USBToolBox` / `UTBMap` are disabled. This approach is
  deprecated on macOS 11.3+ and is a known area for improvement.

## Disclaimer

OpenCore and each kext are covered by their respective upstream licenses. This repository is a
configuration reference. Use at your own risk.
