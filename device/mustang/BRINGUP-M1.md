<!-- SPDX-License-Identifier: MIT -->
# Bring-up runbook: Google Pixel 10 Pro XL (`mustang`) — Milestone M1

Gated on-device runbook to take `mustang` (`laguna` / Tensor G5) from stage 0 to milestone M1:

1. Kernel boots cleanly on slot B (never slot A) with hardware watchdog disabled.
2. USB gadget CDC-NCM (`172.16.42.1`) answers Dropbear SSH in initramfs and OpenSSH after `switch_root`.
3. Storage guard holds (`blockdev --getro` = `1` on slot A, `persist`, `efs`, modem, calibration, boot LUNs `sdb`/`sdc`; `/dev/sda` whole-disk node absent).
4. Framebuffer (`simpledrm`) lights the panel and `omamobile-rootfs` on `userdata` brings up software-rendered Hyprland (`llvmpipe` + `packages/hyprland` `software-renderer.patch`) passing `tools/phone-smoke`.

## Standing safety invariants (never violate)

- **Slot A stays untouched known-good stock Android 17 (`CP3A.261005.005`).** Both slots must carry the exact same bootloader build before starting (`CONTRIBUTING.md` §3).
- **Never run** `fastboot flashall`, `fastboot flashing lock`, `fastboot oem lock`, `fastboot erase`, or `--disable-verity` / `--disable-verification` on `vbmeta`.
- **Never write** any `_a` partition, `super`, `persist`, `efs`, `efs_backup`, `modem_*`, `pvmfw_*`, `devinfo`, or `bootloader_*`.
- **Never set** `BRINGUP_PROBE=1` or an Exynos `SIMPLEFB_TRIG_CON` in `device/mustang/device.conf` (`L-4`): those Exynos MMIO addresses (`0x10060000`, `0x1c0b0000`, `0x1c240030`) do not exist on `laguna`.
- **Never record** real serial numbers, IMEIs, MAC addresses, or proprietary binaries in Git (`tools/pr-check`). Redact serials to `<SERIAL>` in logs.
- **Stop immediately** if any step hits its stop condition; run that step's rollback before diagnosing further.

---

## Pre-flight: Host environment and two-slot stock baseline

- **Risk:** None (read-only checks on a phone already unlocked and flashed to matching stock builds on both slots by its owner).
- **Commands:**
  ```sh
  fastboot getvar product
  fastboot getvar unlocked
  fastboot getvar current-slot
  fastboot getvar version-bootloader
  fastboot getvar battery-voltage
  ```
- **Expect / Pass:** `product: mustang`, `unlocked: yes`, battery comfortably above 50% SoC, and slot A verified able to boot stock Android cleanly (`fastboot --set-active=a && fastboot reboot`, confirm Android UI + USB ADB, then reboot back to bootloader).
- **Stop condition:** Wrong product, locked bootloader, mismatched slot bootloaders, or low battery.
- **Rollback:** Disconnect USB; hold Power to boot existing stock Android on slot A.

---

## Step 1: Read-only audit of live hardware IDs and stock boot headers

Confirm every hardware selector and boot-header parameter against the physical unit before lifting `device/mustang/device.conf` out of stage 0 (`L-44`).

- **Risk:** None (read-only ADB shell commands in stock Android on slot A + host inspection of official Google `mustang-cp3a.261005.005` images).
- **Commands:**
  ```sh
  # 1a. Live hardware stage, board_id/rev, and silicon stepping (in stock Android ADB shell, read-only):
  adb shell 'getprop ro.boot.hardware.revision; getprop ro.boot.cdt_hwid; getprop ro.build.fingerprint'

  # 1b. Inspect official CP3A.261005.005 image headers and AVB footers on the host (in a scratch dir outside Git):
  python3 "$SRC_DIR/system/tools/mkbootimg/unpack_bootimg.py" --boot_img boot.img --out /tmp/mustang-boot
  python3 "$SRC_DIR/system/tools/mkbootimg/unpack_bootimg.py" --boot_img init_boot.img --out /tmp/mustang-init-boot
  python3 "$SRC_DIR/system/tools/mkbootimg/unpack_bootimg.py" --boot_img vendor_boot.img --out /tmp/mustang-vendor-boot
  python3 "$SRC_DIR/system/tools/mkbootimg/unpack_bootimg.py" --boot_img vendor_kernel_boot.img --out /tmp/mustang-vkboot
  python3 "$SRC_DIR/external/avb/avbtool.py" info_image --image boot.img
  python3 "$SRC_DIR/external/avb/avbtool.py" info_image --image init_boot.img
  ```
- **Expect / Pass:**
  - Retail MP unit reports `MP1.0` (`board_id = 0x70506`, `board_rev = 0x10000`); if pre-production (`0x70502`–`0x70505`), update `DTBO_BOARD_ID` / `DTBO_BOARD_REV` in `device.conf` and `mustang-m1-overlay.dts` before step 3.
  - `unpack_bootimg.py` confirms exact page size (`0x1000` vs `0x800`) and whether stock carries `dtb` inside `vendor_kernel_boot.img` or `vendor_boot.img`.
  - `avbtool.py info_image` gives exact partition sizes (`67108864` / `8388608`), `Rollback Index`, and `com.android.build.*.security_patch` for `AVB_FOOTERS`, `AVB_ROLLBACK_INDEX`, and `AVB_SECURITY_PATCH` once PR `#27` merges.
- **Stop condition:** Stock layout differs from `device.conf` (`VENDOR_PAGESIZE`, DTB carrier image, or AVB chaining) before adapting `bin/omamobile` / `device.conf`.
- **Rollback:** `rm -rf /tmp/mustang-*`; phone state untouched.

---

## Step 2: Disable ABL hardware watchdog on unlocked bootloader

Stock Laguna ABL leaves the hardware watchdog active across kernel handoff; without disabling it in fastboot, a mainline kernel resets ~15–30 s after jumping into Linux (`NOTES.md`).

- **Risk:** Low (standard Google fastboot developer toggle on unlocked Pixel 10 bootloaders; reversible with `fastboot oem watchdog enable`).
- **Commands:**
  ```sh
  adb reboot bootloader
  fastboot oem watchdog disable
  ```
- **Expect / Pass:** Fastboot prints `OKAY` with no error.
- **Stop condition:** Command rejected or device enters an unexpected mode.
- **Rollback:** `fastboot oem watchdog enable && fastboot --set-active=a && fastboot reboot`.

---

## Step 3: Build kernel, `ufs-placeholder` DTB, DTBO, and initramfs

 Requires `arch/arm64/boot/dts/google/lga-mustang.dts` (plus `dwc3-google`, `phy-google-usb`, and `ufs@3c400000` when testing persistent rootfs) in `KERNEL_DIR`, PR `#27` merged for `SOC_COMPATIBLE`, and `SOC_COMPATIBLE="b0:0x5:1:0:0 a0:0x5:0:0:0"` uncommented in `device/mustang/device.conf`.

- **Risk:** None (offline host build only; no phone connected required).
- **Commands:**
  ```sh
  DEVICE=mustang bin/omamobile kernel
  DEVICE=mustang bin/omamobile modules
  DEVICE=mustang bin/omamobile initramfs
  DEVICE=mustang bin/omamobile images
  ```
- **Expect / Pass:**
  - `KERNEL_OUT/arch/arm64/boot/dts/google/lga-mustang.dtb` builds cleanly.
  - `cmd_images()` applies `device/mustang/overlays/ufs-placeholder.dtso` (`/aliases/ufs0 = "/ufs-placeholder"`), checks `/soc_compatible`, compiles `mustang-m1-overlay.dtbo` (`0x70506` / `0x10000`), and packs deterministic images with `images/DEVICE` = `mustang`.
  - Decompiling `KERNEL_OUT/images/mustang.dtb` shows `/aliases/ufs0`, `/soc_compatible/B0`, and `compatible = "google,lga-mustang", "google,lga"`.
- **Stop condition:** Any missing Kconfig symbol, missing `/aliases/ufs0`, `/soc_compatible` mismatch, or image packing error.
- **Rollback:** None required (host-only build outputs in `KERNEL_OUT`).

---

## Step 4: Non-persistent RAM boot (`fastboot stage` + `fastboot boot`) — USB SSH & storage guard

Boot the built kernel + initramfs strictly in RAM first so not a single byte of slot A, slot B, or `userdata` is altered until PID 1, USB CDC-NCM (`172.16.42.1`), and the storage guard are proven live.

- **Risk:** Low (volatile RAM boot only; power cycle returns straight to untouched flash). Note: verify USB PHY firmware availability (`usb_phy_fw_2.27.0.bin` in extra initramfs firmware path if `phy-google-usb` requests it before `userdata` mounts).
- **Commands:**
  ```sh
  fastboot --set-active=b
  # Stage the vendor ramdisk/DTB image in RAM and boot Image.lz4 without flashing:
  fastboot stage "$KERNEL_OUT/images/vendor_boot.img"
  fastboot boot "$KERNEL_OUT/images/boot.img"

  # On host once USB NCM interface enumerates (host IP 172.16.42.2/24):
  ssh root@172.16.42.1 '
    uname -a
    tr "\0" "\n" < /proc/device-tree/compatible
    ls -l /dev/sda 2>&1 || true
    for b in /sys/class/block/sd*; do
      n=$(basename "$b")
      p=$(sed -n "s/^PARTNAME=//p" "$b/uevent" 2>/dev/null)
      printf "%-8s %-24s ro=%s\n" "$n" "${p:--}" "$(blockdev --getro "/dev/$n" 2>/dev/null || cat "$b/ro")"
    done
  '
  ```
  *(If ABL requires `vendor_kernel_boot` on slot B rather than `fastboot stage` alone, flash **only** slot B boot partitions via `DEVICE=mustang bin/omamobile flash` while keeping `userdata` untouched, and keep slot A verified clean for one-command rollback.)*
- **Expect / Pass:**
  - `ssh root@172.16.42.1` connects within 15 s and stays up > 120 s (proving hardware watchdog is disabled and `dwc3-google` + `phy-google-usb` hold `usb0`).
  - `/proc/device-tree/compatible` starts with `google,lga-mustang`.
  - If UFS has enumerated: `/dev/sda` whole-disk node **does not exist**, every `sdb*`/`sdc*` boot LUN and every partition outside `boot_b init_boot_b vendor_boot_b vendor_kernel_boot_b dtbo_b userdata` has `ro=1`.
- **Stop condition:** Watchdog reset, panic (`ramoops` at `0x95200000`), no `usb0` link, or any non-allowlisted block device showing `ro=0`.
- **Rollback:** Hold Power + Volume Down (or `echo b > /proc/sysrq-trigger` over SSH/UART while holding Volume Down) -> `fastboot --set-active=a && fastboot reboot`.

---

## Step 5: Live `/chosen` framebuffer inspection and `simpledrm` handoff

Check how stock Laguna ABL passes its pre-initialized splash carveout before touching any DT overlay node.

- **Risk:** Low (read-only inspection over SSH; optional DTBO edit affects only `dtbo_b`).
- **Commands:**
  ```sh
  ssh root@172.16.42.1 '
    ls -la /dev/fb* /dev/dri/* 2>&1 || true
    find /proc/device-tree/chosen /proc/device-tree/reserved-memory -maxdepth 3 2>/dev/null
    dmesg | grep -iE "simple-framebuffer|simpledrm|drm|fb0"
  '
  ```
- **Expect / Pass:**
  - Either `simpledrm` already bound `/dev/dri/card0` + `/dev/fb0` from ABL's `/chosen/framebuffer*` node, **or** `/proc/device-tree/chosen` / `/proc/device-tree/reserved-memory` exposes the exact splash region base, size, width, height (`1080×2404` or `1344×2992`), and `stride` so a clean `simple-framebuffer` node can be added to `device/mustang/mustang-m1-overlay.dts` (`status = "okay"`).
  - Re-check `PANEL_SIZE` in `device/mustang/device.conf` against `cat /sys/class/graphics/fb0/virtual_size`.
- **Stop condition:** Never write guessed MMIO registers to `0x0ec00000` (`sswrp_dpu`) if scanout pauses; investigate `clk_ignore_unused pd_ignore_unused` and VeriSilicon `vs_drm` clock/power domains instead (`L-4`).
- **Rollback:** Revert any experimental edit in `mustang-m1-overlay.dts`; `fastboot --set-active=a && fastboot reboot`.

---

## Step 6: Slot-B install, software-rendered Hyprland desktop, `phone-smoke`, and slot-A rollback check

Install `omamobile-rootfs` built with `PACKAGE=omamobile-device-mustang`, boot into software-rendered Hyprland (`llvmpipe` + `packages/hyprland` `software-renderer.patch`), run `tools/phone-smoke`, and verify one-command rollback to slot A.

- **Risk:** Medium — formats **`userdata`**, which wipes Android user data (back up anything needed first; slot A system/vendor/ calibration partitions remain untouched).
- **Commands:**
  ```sh
  tools/build-packages packages/omamobile-device-mustang
  DEVICE=mustang KERNEL_OUT="$KERNEL_OUT" tools/build-rootfs
  tools/install-phone --device mustang --images "$KERNEL_OUT/images" --dry-run
  tools/install-phone --device mustang --images "$KERNEL_OUT/images"

  # Once booted into Arch on slot B:
  tools/phone-smoke

  # Mandatory rollback gate: prove slot A Android still boots cleanly (note:
  # Android recovery may prompt factory data reset on wiped ext4 userdata if
  # booting Android all the way; verify bootloader + slot switch first):
  tools/phone-ssh 'reboot-to bootloader || echo b > /proc/sysrq-trigger'
  fastboot getvar current-slot
  ```
- **Expect / Pass:**
  - `install-phone --dry-run` prints **only** `boot_b`, `init_boot_b`, `vendor_boot_b`, `vendor_kernel_boot_b`, `dtbo_b`, `userdata`, `--set-active=b`, `reboot`.
  - After `switch_root`, `usb0` holds `172.16.42.1/24`, `/dev/sda` is `0000`, all non-`userdata` partitions stay `ro=1` and hidden (`UDISKS_IGNORE=1`), Hyprland starts on `/dev/dri/card0` (`simpledrm` + `llvmpipe`), and `tools/phone-smoke` exits `0`.
  - Update `docs/DEVICES.md` from stage 0 to milestone M1 **only after** capturing the redacted `tools/phone-smoke` output (`L-28`).
- **Stop condition:** Any failure in `install-phone --dry-run`, `ufs_unguarded` check before `switch_root`, `systemd-networkd` on `usb0`, or `tools/phone-smoke`.
- **Rollback:** In fastboot: `fastboot --set-active=a && fastboot format:f2fs userdata && fastboot reboot` returns the phone completely to stock Android 17 on slot A.
