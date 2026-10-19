---
name: android-phone-modding
description: Use when working on Android phone rooting, bootloader unlocking, flashing, firmware dumps or audio-mod experiments - mtkclient, fastboot/adb, recovery, Magisk, vendor image tooling, per-device gotchas.
---

# Android phone modding

## 1. Recurring workflow (same order on every device)

1. **Probe read-only first.** `adb devices`, `getprop ro.product.device`, `ro.boot.flash.locked`, `settings get global oem_unlock_allowed`; in fastboot `getvar product|unlocked|secure|current-slot|slot-count`, and on MediaTek `fastboot flashing get_unlock_ability` (a bitmask, not yes/no). Record serials: adb and fastboot serial can differ (example LG G4: adb `LGHxxxxxxxx` vs fastboot `xxxxxxxx`; `getvar serial` returned empty).
2. **Check eligibility before attempting anything.** Non-empty `ro.boot.carrierid` = carrier model, OEM unlock permanently hardware-disabled. `unlock_ability` bit 24 (`0x1000000`, `UNLOCK_DISABLED`) = permanently off. `adb root` fails on `user`/`ro.debuggable=0` builds.
3. **Unlock** (`fastboot flashing unlock`, confirm on-device with volume+power), then re-verify with `getvar unlocked` — an `OKAY` is not proof (fake S25 returned `OKAY` then `unlocked: no`). MediaTek route: BROM → `mtk.py da seccfg unlock`.
4. **Back up before writing.** Dump partitions (mtkclient `r`/`rl`, or TWRP booted via `fastboot boot` + `adb pull` when `fastboot fetch` and `adb root` are both unavailable). Keep sha256 next to the image.
5. **Patch** (Magisk-patched boot image, vendor `audio_effects.xml`, privapp allowlist XML) and stage it where the consumer reads it — verify the path first, modules often patch a config the device never loads.
6. **Flash** one partition or one module at a time, one reboot between changes, so a failure has one cause.
7. **Verify with evidence**, not the app's message: `getprop sys.boot_completed` = 1, `su -c id` = uid=0, `magisk -v` matches `*MAGISK:R*`, `grep -c <uuid> /vendor/etc/audio/sku_<sku>/audio_effects.xml`, `grep jamesdsp /proc/$(pidof audioserver)/maps`, `appops get <pkg> PROJECT_MEDIA`.

## 2. Tools and commands actually recorded

### MediaTek / mtkclient (v2.1.4, GPL-3.0)
- Install: `python3 -m venv --system-site-packages .venv`; `.venv/bin/pip install pyusb pycryptodomex pyserial colorama`. Entry points: `mtk`, `stage2`, `da_parser`, `brom_to_offs`, `mtk_gui` (PySide6).
- BROM entry: phone **fully off**, hold volume up, plug USB. USB ids: BROM `0e8d:0003`, preloader `0e8d:2000` ("MT65xx Preloader"). Vendor `0e8d`; udev rule `SUBSYSTEM=="usb", ATTR{idVendor}=="0e8d", MODE="0666", TAG+="uaccess"` in `/etc/udev/rules.d/52-mtk.rules`, then `udevadm control --reload-rules && udevadm trigger`.
- Unlock: `python3 mtk.py da seccfg unlock` (alt `xflash seccfg unlock`); `da seccfg` takes `unlock|lock` plus `--critical`. `stage2 seccfg <flag>` writes `seccfg.bin` — "write seccfg.bin to partition seccfg".
- Dump: `mtk.py r proinfo out.bin --da DA_BR.bin`; `mtk.py devices`; verbs: `printgpt gpt r rl rf rs ro fs w wf wl wo e es ess footer reset meta dumpbrom dumpsram dumppreloader payload crash brute gettargetconfig logs peek stage plstage da script multi`.
- Tanuki/antumbra (Penumbra) unlock flow for Motorola: `antumbra read --da DA_BR.bin <part> <file>` for `preloader`, `preloader_backup`, `oem_mfd`, `proinfo`, `frp`; patch bytes (`oem_mfd@4096=1`, `proinfo@552=1`, `frp` last byte=1); `antumbra x-flash rsc-flash <part> <file> --da DA`; `antumbra write --da DA preloader preloader_raw.img`; `antumbra reboot --da DA`; then fastboot `flashing unlock`, `flash lk_a|lk_b lamu-kaeru.bin`, `reboot-bootloader`, `oem kaeru-version`.
- Gate scripts poll `/sys/bus/usb/devices/*/idVendor` for `0e8d` (and `0e8d:0003` for BROM) before launching the tool.

### fastboot / adb
- Pixel (komodo): `install-grapheneos.sh verify|check|unlock|flash|lock|status`; pin device with `SERIAL=... fastboot -s $SERIAL`, refuse >1 device. Signature: `ssh-keygen -Y verify -f allowed_signers -I contact@grapheneos.org -n "factory images" -s *.zip.sig < zip`, then match the returned key SHA. Requires fastboot ≥ 35.0.1 (installed: 37.0.1).
- GrapheneOS `flash-all.sh`: `flash --slot=other bootloader …img`, `--set-active=other`, `reboot-bootloader`, `flash radio radio-…img`, `erase avb_custom_key` + `flash avb_custom_key avb_pkmd.bin`, `oem uart disable`, `erase fips`, `erase dpm_a dpm_b`, `--disable-super-optimization --skip-reboot update android-info.zip`, `snapshot-update cancel`, then `flash boot init_boot dtbo vendor_kernel_boot pvmfw vendor_boot vbmeta`, `erase userdata metadata`, `flash super` (16 chunks). `android-info.txt`: `require board=komodo`, `version-bootloader`, `version-baseband`, `partition-exists=vendor_kernel_boot`.
- Magisk restore (mi_12_pro `reflash-magisk.sh`): check `getvar product`=`zeus`, `getvar unlocked`=`yes`, `getvar current-slot`, `fastboot flash boot_$slot <img>`, reboot, wait `sys.boot_completed`=1, confirm `magisk -v`.
- Rootless audio app perms: `pm grant <pkg> android.permission.*`, `appops set <pkg> PROJECT_MEDIA allow` (`pm grant` on `PROJECT_MEDIA` throws "Unknown permission" on Android 17; appops still accepts it).

### Magisk / modules
- Module install: `magisk --install-module /data/local/tmp/<zip>`; disable with `touch /data/adb/modules/<m>/disable`; remove with `rm -rf /data/adb/modules/<m>`; global kill switch `adb wait-for-device shell magisk --remove-modules`; safe mode = hold volume-up + power at boot.
- `su` may live at `/debug_ramdisk/su`, not `/system/bin/su` (mi_12_pro) — scripts check both.
- Verify hashes before flashing, as recorded: `MicroGiskGApps-64.zip` `ef901a899bef4083e2a847aff2125e5d94ecc91e7496ac855de79fb8f8b53d63`; `V4A-rootful-arm64.apk` `6855f6510c0acfcaeb7e9fa62121359ed917cf1d1bd4370c9041486f234fef4c`; rootless `ViPER4Android-v4.0.0-rootless-arm64.apk` `e184d90911a0844f315595831def7029b2e34bffe075e69edfeb3a7cc1387db8`; G4 `boot-backup.img` `57e27a154c526349343671ee4f8717108487fe7962ab8904e39582ac6b18b501`.
- Zip packing: `module.prop` at zip root, forward slashes only (Windows `Compress-Archive` backslashes install as inert files), no CR in scripts (a CR makes the watchdog create `disable\r`, which Magisk ignores).

### Audio-effect stack
- Probe path first: `lshal 2>/dev/null | grep -i "audio.effect"` (legacy `android.hardware.audio.effect@7.0::IEffectsFactory/default` = `audio_effects.xml` is read; only `android.hardware.audio.effect.IFactory` = AIDL, XML never read) and `service list | grep android.hardware.audio.effect.IFactory`.
- Find the config the server reads: `grep -rl f27317f4 /system/etc/ /vendor/etc/`; active SKU = `getprop ro.boot.product.vendor.sku` (mi_12_pro: `taro` → `/vendor/etc/audio/sku_taro/audio_effects.xml`).
- Effect identity: implementation UUID `f27317f4-c984-4de6-9a90-545759495bf2`, type `f98765f4-c321-5de6-9a45-123459495ab2`, file `libjamesdsp.so`, config library name `jdsp`, effect name `jamesdsp`.
- Config path in the fragment must stay `/system/lib/soundfx/libjamesdsp.so` — AOSP `checkLibraryPath()` demands that literal substring; `lib64` breaks the load on every device.
- AML must be **off** before installing the engine (`mv /data/adb/modules/aml /data/adb/modules/AML_UIT`): AML v5.1 scans only 32-bit paths, so on arm64 it never sees the library. Re-enable by reversing the `mv`.
- SELinux: if `dlopen` fails, `ls -Z` the lib and `chcon u:object_r:system_lib_file:s0 <path>`, reboot; check `logcat -d -b all | grep -iE "avc.*denied.*(exec|execmod|execmem)"`.
- Pixel 9 series: no `/vendor/etc/audio_effects.xml`, no legacy effect factory, effects hardcoded in `vendor.google.whitechapel.audio.hal.*` — a Magisk audio module cannot work there; the rootless (MediaProjection) app does.

### Image tooling
- GrapheneOS zip unpacked with `bsdtar`/`tar`; `sha256sum`; boot images inspected for `ANDROID!` header (v0, page 4096), `mkbootimg --kernel "$(cat vmlinuz dtb)"` (DTB appended behind the gzip stream), `e2fsck -fn rootfs.img`, `unzip -l` on module zips.
- PostmarketOS device packages are `.apk` (Alpine) tarballs: `linux-postmarketos-qcom-msm8994-6.16-r0.apk` (contains `dtbs/qcom/msm8992-lg-h815.dtb`), `device-lg-h815-2-r0.apk` with `deviceinfo` (`deviceinfo_flash_method="fastboot"`, `deviceinfo_flash_fastboot_partition_rootfs="system"`).

### Partition names seen
- A/B: `boot_a/b`, `lk_a/b` (Moto), `super`, `vendor_boot`, `vendor_kernel_boot`, `init_boot`, `dtbo`, `vbmeta`, `pvmfw`, `radio`/`modem_a`, `avb_custom_key`, `fips`, `dpm_a/b`, `userdata`, `metadata`, `misc`.
- LG G4 (no A/B, no `super`): `boot` = `/dev/block/mmcblk0p38` (40 960 kB), `userdata` = `/dev/block/mmcblk0p50` (23.2 GB), eMMC 4096-byte pages.
- MediaTek read/patch set (tanuki): `preloader`, `preloader_backup`, `oem_mfd`, `proinfo`, `frp`, `misc`.

## 3. Per-device status

| Device | Chipset | Status | What was done | Key gotcha |
|---|---|---|---|---|
| Pixel 9 Pro XL (`komodo`) | not recorded in notes | GrapheneOS flashed, bootloader **relocked**, no root | signature verify → unlock → `flash-all.sh` → `fastboot flashing lock`; rootless V4A installed, appops granted | carrier-id must be empty; AIDL-only audio, so Magisk audio mods never load; verified-boot key hash pinned |
| Xiaomi 12 Pro (`zeus`) | Qualcomm, vendor SKU `taro` (SoC model not recorded) | rooted: LineageOS 23.2 / Android 16, Magisk 30.7, microG + Play Store, JamesDSP/V4A engine **works**; Dolby not installed | module cleanup, privapp-allowlist fix (22→68 perms) + `noprivapprestrict`, SKU audio-config patch, `reflash-magisk.sh` after ROM updates | `su` is `/debug_ramdisk/su`; AML must be off first; module touching power in first 3 s of boot disables everything |
| Moto G15 Power (`lamu`) | MediaTek Helio G81 Extreme | **unlock not completed**: antumbra/tanuki `Failed to read preloader` twice (`Error: Timeout` after handshake, 2026-10-06 19:10 & 19:12); mtkclient `r proinfo` scripts written, no successful dump recorded; ROM/partition images downloaded (`dl.log`: boot, dtbo, init_boot, lk, vbmeta, vendor_boot, build-manifest, rc=0) | udev rule, BROM/0e8d gate + USB enumeration loggers, mtkclient read scripts, tanuki unlock tool run | preloader enumerates as `0e8d:2000`, handshake completes then times out; BROM `0e8d:0003` needs the phone held physically |
| LG G4 (`h815`) | MSM8992 (Snapdragon 808) | mainline Arch/omarchy image **built and verified, never booted**; `boot` flashed then restored, phone back on crDroid Android, bootloader `unlocked: yes`, no root | `backup-boot.sh` (`fastboot boot` TWRP + `adb pull`, read-only), `flash-userdata-twrp.sh` streams raw rootfs in 256 MiB slices to `dd conv=notrunc`, `flash.sh boot-test` pending | no A/B → no fallback after `flash boot`; `fastboot` refuses `userdata`/`dtbo` writes; adb ≠ fastboot serial; DTB has no display/GPU/audio/wifi nodes (headless) |
| Fake "Galaxy S25" (`<serial>`) | MediaTek MT6735, Android 8.1, armeabi-v7a | **root impossible**: permanently locked, all software routes dead | read-only scan, APK static read (`/system/app/Rxd/Rxd.apk`, sha256 `cd730f5a…bd1756f`, signed `CN=demo` MediaTek demo key), route enumeration | `fastboot flashing unlock` → `OKAY` but `getvar unlocked: no`, `unlock_ability = 16777216`; partition flash → `FAILED (remote: 'not allowed in locked state')` |
| OnePlus 6T (`fajita`, in lg_g4 notes) | Snapdragon 845 (SDM845) | boots Omarchy 4 on `linux-postmarketos-qcom-sdm845`; WiFi, mobile data, SMS work; call audio partial, Bluetooth no | replaced one A/B boot slot with mainline image, `--set-active=b` | Android system partitions untouched (only boot slot + userdata); no RTC, suspend never resumes, omarchy-update clobbers local files |

## 4. Warnings actually noted

- **Wipes:** on the Pixel, `unlock`, `flash` and `lock` each print "WIST ALLE GEGEVENS … geen terugdraaiversie"; tanuki README: "unlocking the bootloader wipes all data". Back up first.
- **`OKAY` ≠ unlocked.** Always re-read `getvar unlocked` and `flashing get_unlock_ability`; partition flashes fail with `not allowed in locked state` while locked.
- **GrapheneOS:** an unlocked bootloader means "incomplete installation", no support, every OTA must be re-patched and re-signed with your own key (`avb_custom_key` + Custota) or updates silently stop; original Magisk is "definitief kapot" there. The setup-wizard checkbox that disables OEM unlocking must be left alone — ticking it is the classic lock-out, notes say hard-brick risk.
- **LG G4 has no fallback slot:** `flash.sh` refuses to run until `boot-backup.img` exists; `flash boot` is the irreversible step, run `boot-test` (`fastboot boot`, writes nothing) first. `fastboot erase dtbo` / `flash userdata` fail with `cannot erase/flash this partition in unlocked state`.
- **MediaTek BROM work needs physical hands:** `adb reboot` reaches fastboot, not BROM; the phone must be off with volume up held while plugging in.
- **Don't touch the phone during boot** — the `abootloop` module disables all modules if the power button is pressed in the first 3 seconds (looks like a dead/empty safe mode; distinguish from a microG bootloop by `cat /proc/uptime`).
- **Audio-module bootloop** (audio server crashing every boot) → `adb wait-for-device shell magisk --remove-modules`, safe mode, or `rm -rf /data/adb/modules/<m>` from recovery. The rv4a module's `post-fs-data.sh` moves aside any config that fails validation.
- **Verify before flashing:** sha256 for every artifact; GrapheneOS sig must verify *and* match the documented key, else "NIET flashen". Never install two libraries claiming the same effect UUID; Dolby Atmos ports come from anonymous hosts with no hash (notes advise against).
- **ROM updates undo work:** a LineageOS update disables Magisk (rerun `reflash-magisk.sh`) and drops microG's privapp allowlist (rerun `fix-microg-privapp.sh`).
- **Unverified in the notes:** mtkclient `devices` listing does not prove a ROM is supported (`mt6735` sits in `brom_config.py` with `DAmodes.LEGACY`, but `mtk.py devices` showed only a Wiko Lenny 4); whether the G4 bootloader passes the appended DTB to the kernel was never confirmed.
