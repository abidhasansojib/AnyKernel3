# AnyKernel3 (GKI 2.0 Edition)

> Flashable Zip Template for Android Generic Kernel Images (GKI) & Custom Kernels  
> Maintained by **@abidhasansojib** • Upstream Base by **osm0sis @ xda-developers**

---

## 📖 Overview

This edition of **AnyKernel3** is specifically optimized for modern Android **GKI (Generic Kernel Image)** devices (Kernels `5.10`, `5.15`, `6.1`, `6.6`, `6.12` and newer). It features full 64-bit (AArch64) native static binaries, automatic A/B slot detection, header v3/v4 multi-partition support, runtime GKI validation, and root preservation (KernelSU / Magisk).

---

## ⚡ How It Works (Execution Flow)

```
┌──────────────────────────────────────────────────────────┐
│ 1. Recovery executes META-INF/update-binary & banner     │
└────────────────────────────┬─────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────┐
│ 2. anykernel.sh defines properties & sources ak3-core.sh │
└────────────────────────────┬─────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────┐
│ 3. setup_ak detects active slot (_a/_b) & boot partition │
└────────────────────────────┬─────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────┐
│ 4. GKI Version Check (/proc/version: 5.10 - 6.12+)       │
└────────────────────────────┬─────────────────────────────┘
                             ▼
┌──────────────────────────────────────────────────────────┐
│ 5. split_boot dumps and inspects the device boot image   │
└────────────────────────────┬─────────────────────────────┘
                             ▼
             ┌───────────────┴───────────────┐
             │ Ramdisk present in boot.img?   │
             └───────┬───────────────┬───────┘
                     │ YES           │ NO (Header v4 / init_boot)
                     ▼               ▼
         ┌───────────────────┐ ┌───────────────────┐
         │ unpack_ramdisk    │ │ flash_boot        │
         │ write_boot        │ │ (Direct kernel    │
         │ (Repack & flash)  │ │  Image flash)     │
         └───────────────────┘ └───────────────────┘
```

1. **Initialization & Banner**: The recovery engine launches `update-binary`, which loads and prints the stylized ASCII `banner`.
2. **Setup & Slot Resolution (`setup_ak`)**: Automatically identifies whether the device is A/B partitioned and locates the corresponding target block partition (e.g. `/dev/block/by-name/boot_a`).
3. **GKI Branch Validation**: Reads `/proc/version` to verify that the active system is a supported GKI branch (`5.10*`, `5.15*`, `6.1*`, `6.6*`, `6.12*`). If an incompatible kernel version is detected, execution safely aborts.
4. **Boot Splitting (`split_boot`)**: Dumps the current boot image to disk and parses its header format using 64-bit `magiskboot`.
5. **Smart Dispatching**:
   - **Boot with Ramdisk**: If `ramdisk.cpio` is present in `boot.img`, it unpacks the ramdisk, integrates the new kernel image, handles Magisk/KernelSU preservation, repacks, and flashes (`write_boot`).
   - **Boot without Ramdisk (Modern GKI / Header v4)**: On Android 13+ devices where the first-stage ramdisk is located in `init_boot`, AnyKernel3 directly executes `flash_boot` to replace the kernel image in the `boot` partition without disturbing the ramdisk or dm-verity state.

---

## 🚀 User Manual: How to Build & Flash

### 1. Place Your Kernel Build
Copy your compiled kernel binary into the root of the AnyKernel3 folder:
- **Kernel Image**: `Image`, `Image.gz`, `Image-dtb`, etc.
- *(Optional)* **Auxiliary Images**: If your build produces separate partitions, place `dtb`, `dtbo.img`, `vendor_dlkm.img`, or `system_dlkm.img` in the root directory.

### 2. Configure `anykernel.sh` (Optional)
Edit `anykernel.sh` to customize your kernel branding or adjust settings:
```bash
properties() { '
kernel.string=My Custom GKI Kernel by Developer
do.devicecheck=0
do.cleanup=1
do.check_boot_version=0
'; }
```

### 3. Package the Flashable Zip
Run the following command from the root of the AnyKernel3 directory:
```bash
zip -r9 UPDATE-AnyKernel3.zip * -x .git README.md "*placeholder"
```

### 4. Flash to Device
Transfer `UPDATE-AnyKernel3.zip` to your device and flash it using:
- **Custom Recovery**: TWRP, OrangeFox, PBRP, etc.
- **Root Flashing Utilities**: [Kernel Flasher](https://github.com/fatalcoder524/KernelFlasher).

---

## ⚙️ Properties & Variables Reference

### Properties (`anykernel.sh`)

| Property | Default | Description |
| :--- | :---: | :--- |
| `kernel.string` | `GKI KERNEL` | Kernel name displayed in the recovery console during flash. |
| `do.devicecheck` | `0` | Enable device codename verification against `device.name1..5`. |
| `device.name1..5` | *(empty)* | Allowed device codenames (e.g. `cheetah`, `husky`). |
| `do.modules` | `0` | Push `.ko` modules from `/modules` to vendor partitions. |
| `do.systemless` | `0` | Install kernel modules systemlessly via KernelSU/Magisk helper module. |
| `do.cleanup` | `1` | Clean up temporary files in `/tmp` upon successful flash. |
| `do.cleanuponabort` | `0` | Clean up `/tmp` workspace if installation aborts. |
| `do.check_boot_version` | `0` | Validate kernel version string compatibility before flashing. |
| `keycheck.timeout` | `10` | Timeout in seconds for interactive volume key input prompts. |

### Shell Variables (`anykernel.sh`)

| Variable | Default | Description |
| :--- | :---: | :--- |
| `BLOCK` | `boot` | Target partition to flash (e.g. `boot`, `init_boot`, `vendor_boot`, or auto). |
| `IS_SLOT_DEVICE` | `auto` | Enable automatic A/B slot suffix detection (`_a`/`_b`). |
| `RAMDISK_COMPRESSION` | `auto` | Repack compression format (`auto`, `none`, `gz`, `lz4-l`, `xz`). |
| `PATCH_VBMETA_FLAG` | `auto` | Manage AVBv2 vbmeta flag patching (`auto`, `0`, `1`). |
| `NO_MAGISK_CHECK` | `1` | Skip Magisk-specific kernel repatching on modern GKI kernels. |

---

## 🧰 Built-in 64-Bit Toolchain

All binaries in `tools/` are statically compiled for **AArch64 (ARM64)** to ensure seamless compatibility with 64-bit-only Android environments:
- **`magiskboot`**: 64-bit image unpacking, repacking, decompression, and hexpatching.
- **`busybox`**: Multi-call binary providing complete shell utilities.
- **`getevent`**: Direct Linux input event reader for hardware volume-key navigation.
- **`httools_static` & `lptools_static`**: Logical partition (Dynamic Partition / `super`) mapping and resizing.
- **`snapshotupdater_static`**: Virtual A/B snapshot update handler.
- **`fec`**: Forward error correction utility for partition verification.

---

## 🛠️ Command Methods Reference (`ak3-core.sh`)

| Command Method | Description |
| :--- | :--- |
| `ui_print "<text>"` | Prints text directly to recovery terminal output. |
| `abort "<text>"` | Displays error message and stops flashing immediately. |
| `split_boot` | Dumps and unpacks boot image headers without extracting ramdisk. |
| `unpack_ramdisk` | Extracts `ramdisk.cpio` into the `$RAMDISK` directory. |
| `unpack_vendorrd <name>` | Unpacks a specific sub-ramdisk from `vendor_boot` header v4. |
| `repack_ramdisk` | Repacks modified `$RAMDISK` into `ramdisk-new.cpio`. |
| `flash_boot` | Builds, signs, and writes repacked boot image to the target block. |
| `flash_generic <partition>` | Automatically maps and flashes an image to a logical/physical partition (e.g. `vendor_dlkm`, `dtbo`). |
| `write_boot` | Standard pipeline executing `repack_ramdisk` followed by `flash_boot`. |
| `backup_file <file>` | Creates a backup copy (`file~`) prior to patching. |
| `restore_file <file>` | Restores a backed up file. |
| `replace_string <file> <if> <orig> <rep>` | Replaces string occurrence in text/rc files. |
| `patch_cmdline <name> <replacement>` | Injects or modifies kernel command-line parameters. |
| `patch_fstab <fstab> <mount> <fs> ...` | Updates mount flags and options in fstab tables. |

---

## 📄 License

This project is licensed under the GNU General Public License as detailed in the [LICENSE](LICENSE) file.
