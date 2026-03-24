# TWRP Device Tree — Nubia Red Magic 11 Pro (NX809J)

Unofficial TWRP recovery for the Nubia Red Magic 11 Pro (NX809J).

## Status

- [x] Device tree created with verified hardware values
- [x] Dynamic partitions configured (19.3 GB super)
- [x] A/B partition layout confirmed with dedicated recovery partition
- [x] Vendor blobs extracted (43 files: crypto HALs + dependencies)
- [x] Touch driver modules included (Synaptics TCM via zte_tpd)
- [ ] Builds successfully via GitHub Actions
- [ ] Boots to recovery UI
- [ ] Touch input functional
- [ ] /data decryption (FBE with wrappedkey_v0)

## Device Specifications

| Component | Detail |
|-----------|--------|
| Device | Nubia Red Magic 11 Pro |
| Codename | canoe (NX809J) |
| SoC | Qualcomm Snapdragon 8 Elite (SM8750) |
| SoC Platform | sun |
| GPU | Adreno 830 |
| Architecture | arm64 (ARMv8-A) |
| Kernel | Linux 6.12.23 (GKI, Android 16) |
| Boot header | v4 |
| RAM | 12/16 GB |
| Display | AMOLED, DRM DSI (card0-DSI-1) |
| Touch | Synaptics TCM (via zte_tpd kernel module) |
| Partition scheme | A/B with dedicated recovery partition |
| Super partition | 19,327,352,832 bytes (~18 GB) |
| Userdata | f2fs, FBE (AES-256-XTS, wrappedkey_v0, inlinecrypt) |
| Metadata | f2fs |
| Crypto HAL | KeyMint SPU (not Keymaster 4.x) |
| USB controller | a600000.dwc3 |

## Partition Layout

This device uses a **dedicated recovery partition** — not recovery-in-boot or recovery-in-vendor_boot.

Physical partitions (A/B):
`boot`, `dtbo`, `init_boot`, `recovery`, `vendor_boot`, `vbmeta`, `vbmeta_system`, `modem`, `bluetooth`, `dsp`, `persist`, `ztecfg`

Dynamic (logical) partitions inside super:
`system`, `system_ext`, `product`, `vendor`, `vendor_dlkm`, `system_dlkm`, `odm`

## Included Vendor Blobs

### Crypto HAL Binaries (5)
- `qseecomd` — TEE communication daemon
- `android.hardware.security.keymint-service-spu-qti` — KeyMint (SPU-backed)
- `android.hardware.gatekeeper-rust-service-qti` — Gatekeeper (Rust, SPU-backed)
- `android.hardware.weaver-service-spu-qti` — Weaver (credential storage)
- `android.hardware.health-service.qti` — Health HAL

### Shared Libraries (36)
Full dependency chain for decryption, including `libQSEEComAPI.so`, `libspukeymint.so`, `libcrypto.so`, `libdrmfs.so`, and 32 others. All extracted from stock vendor partition with dependencies verified via `readelf`.

### Kernel Modules (4)
- `zte_tpd.ko` — Touch panel driver (Synaptics TCM)
- `panel_event_notifier.ko` — Display event notifier (zte_tpd dependency)
- `haptic_86938.ko` — Haptic feedback driver
- `ifas.ko` — Input filter

## Build Instructions

### Via GitHub Actions (recommended)
1. Fork this repository
2. Go to Actions tab → Build TWRP Recovery
3. Click "Run workflow"
4. Download `recovery.img` from Releases when complete (~25 min total)

### Local Build
```bash
mkdir ~/twrp && cd ~/twrp

# Init TWRP manifest (LineageOS-based, Android 14.1)
repo init --depth=1 \
    -u https://github.com/minimal-manifest-twrp/platform_manifest_twrp_aosp.git \
    -b twrp-14.1
repo sync -j$(nproc) --force-sync

# Clone device tree
git clone https://github.com/cmfnels/device_nubia_NX809J -b main device/nubia/NX809J

# Build
export ALLOW_MISSING_DEPENDENCIES=true
export LC_ALL="C"
source build/envsetup.sh
lunch twrp_NX809J-ap2a-eng
make recoveryimage -j$(nproc)
```

Output: `out/target/product/NX809J/recovery.img`

## Flashing

**Always test-boot first!**

```bash
# Test (safe — nothing written to device)
adb reboot bootloader
fastboot boot recovery.img

# Install permanently (only after confirming test-boot works)
fastboot flash recovery_a recovery.img
fastboot flash recovery_b recovery.img
fastboot reboot
```

## Debugging

### Get recovery log
```bash
adb shell cat /tmp/recovery.log > recovery_log.txt
```

### If screen is black
- Try different pixel format in BoardConfig.mk: `BGRA_8888` or `RGBA_8888`
- Check ADB: `adb devices` may still show recovery device

### If no touch
- Check if zte_tpd loaded: `adb shell lsmod | grep zte_tpd`
- May need `kmparam.ko` dependency (built into vendor_boot ramdisk)

### If /data not decrypted
- Check recovery.log for SELinux denials (`avc: denied`)
- Check if qseecomd/keymint started: `adb shell ps | grep qsee`
- Temporary debug: add `TW_FORCE_SELINUX_PERMISSIVE := true` to BoardConfig.mk

## Known Limitations

- `kmparam.ko` (touch driver dependency) is loaded from vendor_boot ramdisk at runtime — not available as standalone .ko file. Touch may require manual module loading if it's not kernel-builtin.
- SELinux `recovery` context may not have sufficient permissions for SPU communication. Decryption debugging may require permissive mode initially.
- This is the first custom recovery for the NX809J. Expect rough edges.

## Hardware Values Source

All partition sizes, HAL versions, filesystem types, and device paths were verified directly from a running Red Magic 11 Pro via `getprop`, `/vendor/etc/fstab.qcom`, `readelf`, `lsmod`, and `/proc/bus/input/devices`.

## Contributing

Contributions welcome, especially for:
- Decryption debugging and SELinux policy fixes
- Touch driver improvements
- Testing on Red Magic 11 Pro+ variants
- OrangeFox port based on this tree

## Disclaimer

**⚠️ USE AT YOUR OWN RISK**

This is an unofficial build in early development. Always test with `fastboot boot` before permanent flashing. The developers are not responsible for bricked devices, broken functionality, or voided warranties.

## Credits

- Device tree and vendor blob extraction by [@cmfnels](https://github.com/cmfnels)
- TWRP by [TeamWin](https://twrp.me/)
- Reference: [kmiit/twrp_device_oplus_sm87xx](https://github.com/kmiit/twrp_device_oplus_sm87xx) (OnePlus 13 SM8750 TWRP)
