# Pixel 3 (blueline) + LineageOS 22.2 — kernel root with SukiSU-family (BakaSU) + SUSFS

## Why not SukiSU Nova directly

The Pixel 3 kernel is `4.9.337-gdf5cccf77f2a` (read from `boot.img`; the hash is 13 characters). Facts from the SukiSU project:

- SukiSU Nova (`SukiSU-Nova/SukiSU-Nova`, release v3.0.0, site `sukisu.org`) supports **GKI 2.0 only** for LKM mode. LKM modules ship for `android12-5.10` and newer. There is no 4.9 module, so the Manager "Tap to install" card cannot do anything on this device.
- Self-built kernels need **kernel 4.14 or newer**. The kernel source has no code path for 4.9.
- Older kernels work with **BakaSU** (`Baka-SU/BakaSU`, formerly ReSukiSU, a fork of SukiSU/Nova). It supports non-GKI down to 3.4 with manual hooks, and SUSFS backport down to 4.3.

So this workflow builds the LineageOS 22.2 kernel from source with BakaSU + SUSFS, then packs a flashable `boot.img`.

## What is already prepared in this fork

| File | Purpose |
|---|---|
| `.github/workflows/build-google-pixel-3-blueline-lineageos22-a15.yml` | New build workflow for blueline |
| `boot/lineage-22.2-blueline-boot.img` | Stock LineageOS boot image, used as packing template |

Workflow values that matter:

| Setting | Value | Reason |
|---|---|---|
| `KERNEL_SOURCE` / `KERNEL_BRANCH` | `LineageOS/android_kernel_google_msm-4.9` / `lineage-22.2` | Same tree as the ROM |
| `DEFCONFIG_NAME` | `b1c1_defconfig` | blueline + crosshatch |
| `CLANG_SOURCE` | LineageOS prebuilt clang r416183b | Same compiler as the ROM |
| `KERNELSU_AUTO_FORK` | `bakasu` | Works on 4.9; `sukisu` would fail |
| `SUSFS_ENABLE` | `true` | Root hiding |
| `SUSFS_FOLDER_FIXED` | `pixel_3_series_aosp` | Ready-made 4.9 SUSFS patch set |
| `HOOK_METHOD` | `syscall` | Manual hook; tracepoint hook needs GKI 2.0 |
| `PACK_METHOD` | `MKBOOTIMG` | Pixel 3 has no usable TWRP on Android 15, so an AnyKernel3 zip cannot be flashed |
| `KPM_ENABLE` | `false` | KPM on 4.9 needs an extra `set_memory` backport |
| `BOOT_SOURCE` | `.../boot/lineage-22.2-blueline-boot.img` | Template for the new boot image |

The workflow copies the exact release string of the stock boot image (parsed from its `vermagic`) into `CONFIG_LOCALVERSION`. The vendor kernel modules (`wlan.ko`, the audio modules) check `vermagic` and symbol CRCs, so a changed string or a different config shape breaks Wi-Fi and other drivers.

## Why Wi-Fi and camera keep working

| Driver | Where it lives | Effect of this build |
|---|---|---|
| Camera (`CONFIG_SPECTRA_CAMERA=y`) | Inside the kernel image | Rebuild from the same sources and config; no module involved |
| Wi-Fi (`CONFIG_QCA_CLD_WLAN=m`, external `qcacld-3.0`) | Stock `wlan.ko` on `/vendor/lib/modules` | Loads only if `vermagic` and symbol CRCs match |

The build reproduces the ROM kernel ABI in four ways:

1. **Same config shape.** The released kernel is built with `build.config.bluecross`: `CONFIG_LTO=y`, `CONFIG_LTO_CLANG=y`, `CONFIG_CFI=y`, `CONFIG_CFI_CLANG=y`, `CONFIG_CFI_CLANG_SHADOW=y`, `CONFIG_LD_DEAD_CODE_DATA_ELIMINATION=y`, `CONFIG_THIN_ARCHIVES=y`, with `CONFIG_CFI_PERMISSIVE` off. The workflow enables the same set. LTO and CFI **must** stay on: the stock `wlan.ko` is CFI-instrumented and imports `__cfi_slowpath`, so a non-CFI kernel cannot load it and Wi-Fi dies. `build.config.bluecross_no-cfi` exists in the tree but the shipped image does not use it. The final vmlinux is a full-LTO gold link (`ld.gold` + `LLVMgold.so`), so the workflow adds 16 GB of swap before the build.
2. **Same release string.** The workflow unpacks the stock `boot.img`, reads its `vermagic`, and copies the release string into `CONFIG_LOCALVERSION`:
   ```
   stock vermagic: 4.9.337-gdf5cccf77f2a SMP preempt mod_unload modversions aarch64
   CONFIG_LOCALVERSION: -gdf5cccf77f2a
   ```
   The value is derived from the image, never hardcoded, so it stays correct after a LineageOS OTA. Note the hash has 13 characters: `git rev-parse --short=12` returns 13 in this repository because 12 is ambiguous.
   `CONFIG_LOCALVERSION_AUTO` is disabled and `.scmversion` is removed, because `scripts/setlocalversion` in this tree also walks nested git repositories (KernelSU, NoMount, patches) and appends their revisions, which overflows the 64-character limit.
3. **Same symbol CRCs.** `CONFIG_MODVERSIONS=y` is kept, and every change in this build touches `.c` files only (SUSFS patch, syscall hook patch, KernelSU driver). No struct or header change, so CRCs stay identical.
4. **No module signing.** `CONFIG_MODULE_SIG` is off in the defconfig, so the stock unsigned `wlan.ko` is accepted.

The workflow step **"🧪 Verify module ABI matches the ROM"** fails the build if the built release string differs from the stock one, if `CONFIG_MODVERSIONS`, `CONFIG_SMP`, `CONFIG_PREEMPT`, `CONFIG_MODULE_UNLOAD` are missing, if `CONFIG_MODULE_SIG` is on, if `CONFIG_KSU` is off, or if the compiled `Image` does not contain the release string. A green run means the stock `wlan.ko` loads.

The shared pack step only accepts `Image`, `Image.gz` or `Image.*-dtb`. This device builds `Image.lz4` with a separate dtb in the boot header, so the workflow packs the image itself: it takes `ramdisk` and `dtb` from the stock template, drops in the new `Image.lz4`, and writes a plain `.img`. Result layout is identical to the LineageOS release image.

## Steps

### 1. Check the device state

```sh
adb shell uname -r
adb shell "zcat /proc/config.gz | grep -E 'CONFIG_KALLSYMS|CONFIG_MODVERSIONS'"
adb reboot bootloader
fastboot getvar current-slot
fastboot getvar unlocked
```

Expect `4.9.337-...`, `CONFIG_KALLSYMS_ALL=y`, `CONFIG_MODVERSIONS=y`, `unlocked: yes`.

Keep a copy of the stock `boot.img` (already in `boot/`) for rollback.

### 2. Fork and branch

Done already: fork `aaronmurniadi/NonGKI_Kernel_Build_2nd`, branch `blueline`. `BOOT_SOURCE` points at that branch:

```
https://raw.githubusercontent.com/aaronmurniadi/NonGKI_Kernel_Build_2nd/blueline/boot/lineage-22.2-blueline-boot.img
```

If you rename the fork, the branch, or the image path, update `BOOT_SOURCE` in the workflow. If you move the image out of git (for example to a release asset), keep the URL reachable without login.

### 3. Disable the weekly mass build

Done already. `build-release.yml` (Wednesday schedule, all 19 devices) and `build-kernel.yml` (matrix) are disabled in the fork. Only the blueline workflow stays active.

### 4. Run the workflow

A run is already started on branch `blueline`:

https://github.com/aaronmurniadi/NonGKI_Kernel_Build_2nd/actions/runs/44588245749

Actions → "Build Google Pixel 3 (LineageOS 22.2 A15)" → Run workflow → branch `blueline`.

Expect 40–90 minutes on a free runner. The artifact is a single file named `blueline-lineageos22_2_a15-<timestamp>-image.img`.

### 5. Flash

Verification (AVB) must be off. LineageOS install already disables it on most Pixel 3 units, but run this once to be sure:

```sh
fastboot --disable-verity --disable-verification flash vbmeta vbmeta.img
```

Then flash the new kernel to both slots:

```sh
fastboot flash boot_a blueline-lineageos22_2_a15-<timestamp>-image.img
fastboot flash boot_b blueline-lineageos22_2_a15-<timestamp>-image.img
fastboot reboot
```

Test boot only (temporary, reverts on reboot):

```sh
fastboot boot blueline-lineageos22_2_a15-<timestamp>-image.img
```

### 6. Install the manager

Download `ReSukiSU_v4.2.0-*_arm64-v8a-release.apk` from `Baka-SU/BakaSU` releases and install it. The manager and the kernel must come from the same project.

### 7. Verify

```sh
adb shell uname -r                     # 4.9.337-g... (unchanged string)
adb shell getprop | grep -i wifi       # Wi-Fi driver loaded from vendor
```
Open the manager: it must show "Working" and the kernel version. Grant root to `su` in a terminal.

### 8. After a LineageOS OTA

Every OTA overwrites `boot.img`. Root is lost after an update. Reflash the built image to the updated slot:

```sh
adb reboot bootloader
fastboot flash boot_a <image>
fastboot flash boot_b <image>
```

If the OTA moves the kernel commit, rebuild from the new branch head so the release string matches again. A mismatch breaks vendor modules.

## If something breaks

| Symptom | Cause | Fix |
|---|---|---|
| Boot loop | Patch or config failure | `fastboot flash boot_a boot/lineage-22.2-blueline-boot.img` (repeat for `boot_b`) |
| Boots, no Wi-Fi | Stock `wlan.ko` rejected: `vermagic` or CRC mismatch | Check `adb shell dmesg \| grep -i wlan`. Rebuild with the ABI guard step green |
| Boots, no camera | Camera HAL failed | `adb logcat \| grep -i cameraserver`. Restore the stock boot image and report |
| "Your device is corrupt" warning | AVB mismatch on boot | `fastboot --disable-verification flash vbmeta vbmeta.img` |
| Workflow fails at "Verify module ABI" | The template boot image does not match the ROM anymore (LineageOS OTA), or LTO/CFI got disabled again | Re-extract `boot/lineage-22.2-blueline-boot.img` from the installed OTA; the release string is derived from it automatically. Keep `CONFIG_LTO_CLANG`/`CONFIG_CFI_CLANG` enabled. |
| Workflow fails in SUSFS patch step | Patch context drift in the LineageOS tree | Read the patch log, or pin an older kernel commit |
| Link step killed (OOM) | LTO on a 7 GB runner | Already disabled in the workflow; if it still fails, set `FREE_MORE_SPACE: "true"` |
