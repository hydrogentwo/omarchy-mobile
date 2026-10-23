<!-- SPDX-License-Identifier: MIT -->
# Google Pixel 10 Pro XL (`mustang`)

Stage-0 hardware reference, kernel Delta from Exynos Tensor (`gs101`–`zumapro`), and cited bring-up facts for the Google Pixel 10 Pro XL (`mustang`, Tensor G5 / `laguna`).

Every statement below carries a verification tag:

- `[V]` — directly verified against an explicitly cited public source in [Sources](#sources).
- `[I]` — inferred from adjacent Tensor generations or partial upstream code and must be verified on live `mustang` hardware before acting on it.

No on-phone boot has been executed in this repository yet (`docs/DEVICES.md` stays at stage 0 until [`BRINGUP-M1.md`](BRINGUP-M1.md) completes on hardware and `tools/phone-smoke` evidence is captured).

## Identity and stock baseline

| Property | Value | Status / source |
| --- | --- | --- |
| Marketing name | Google Pixel 10 Pro XL | `[V]` `F/images`, `U/lga-dts` |
| Codename / board family | `mustang` (shared family repo `muzel`: `frankel`, `blazer`, `mustang`, `rango`) | `[V]` `A/muzel` (`BoardConfig-common.mk`), `U/lga-binding` |
| Regulatory / SKU models | `GUL82` (`ro.boot.hardware.sku=GUL82`) | `[V]` `A/mustang` (`init_mustang.cpp`) |
| SoC | Google Tensor G5 (`laguna`, upstream prefix `lga`) — TSMC N3E, non-Exynos architecture | `[V]` `U/lga-soc`, `U/lga-dts`, `A/laguna` |
| CPU | 8 cores (`aarch64`): Cortex-X4 + Cortex-A725 + Cortex-A520 (`cortex-a55` baseline ABI in Android) | `[V]` `A/laguna` (`BoardConfig-common.mk`) |
| GPU | Imagination PowerVR `DXT-48-1536` (`34f00000.gpu0`, Volcanic architecture, BVNC `71.2.2448.1212`) | `[V]` `A/laguna` (`sepolicy/vendor/genfs_contexts`, `proprietary-files-vendor.txt`) |
| Display panel | 6.8" LTPO OLED, native `1344 × 2992`, default stock boot mode `1080 × 2404 @ 120 Hz` | `[V]` `A/mustang`, `A/laguna` (`vendor.prop`) |
| Root DT `compatible` | `"google,lga-mustang", "google,lga"` | `[V]` `U/lga-binding`, `U/lga-dts` |
| Reference crystal `osc` | `38,400,000 Hz` (`38.4 MHz`, vs `24.576 / 26 MHz` on Exynos Tensor) | `[V]` `U/lga-dts` |
| Stock baseline build | Android 17 `CP3A.261005.005` (Oct 2026), GKI `6.6.127` (`os_version = 17.0.0`, `os_patch_level = 2026-10`) | `[V]` `F/images`, `F/ota`, `A/muzel` |
| Stock factory archive | `mustang-cp3a.261005.005-factory-099acc44.zip` (`sha256: 099acc44e2f87fe2735d92952081d64550ff08b4a6390bb6b001c8451695657e`) | `[V]` `F/images` |
| Stock full OTA archive | `mustang-ota-cp3a.261005.005-62f7ec41.zip` (`sha256: 62f7ec41d7ed3b945db37cdd211fb6314c1aae5ef060d8141e397571eeb85`) | `[V]` `F/ota` |

## Hardware revisions and bootloader matching

### Board IDs (`DTBO_BOARD_ID` / `DTBO_BOARD_REV`)

Stock ABL selects the matching entry from `dtbo` using root `board_id` and `board_rev` `[V]`:

| Hardware stage | `board_id` | `board_rev` | Source |
| --- | --- | --- | --- |
| Proto 1.0 / 1.1 | `0x70502` | `0x10000` / `0x10100` | `[V]` `A/mustang` (`init_mustang.cpp`), `U/lga-dts` |
| EVT 1.0 / 1.1 / Wingboard | `0x70503` | `0x10000` / `0x10100` | `[V]` `A/mustang` (`init_mustang.cpp`) |
| DVT 1.0 | `0x70504` | `0x10000` | `[V]` `A/mustang` (`init_mustang.cpp`) |
| PVT 1.0 | `0x70505` | `0x10000` | `[V]` `A/mustang` (`init_mustang.cpp`) |
| Retail MP 1.0 | `0x70506` | `0x10000` | `[V]` `U/lga-dts` (`lga-mustang.dts`), `A/mustang` |

`device/mustang/device.conf` and `device/mustang/mustang-m1-overlay.dts` target retail MP 1.0 (`board_id = <0x70506>`, `board_rev = <0x10000>`). [`BRINGUP-M1.md`](BRINGUP-M1.md) step 1 checks the unit's actual `ro.boot.hardware.revision` / `cdt_hwid` before booting any custom image so a PVT (`0x70505`) or DVT (`0x70504`) unit never mismatches `dtbo`.

### Base DTB directory and SoC selector (`google/`, `/soc_compatible`)

Tensor G5 (`laguna` / `lga`) breaks cleanly from Samsung Exynos (`gs101`, `gs201`, `zuma`, `zumapro`):

1. **Upstream directory convention (`arch/arm64/boot/dts/google/`):** Mainline Linux v7.4 added `CONFIG_ARCH_GOOGLE` (`arch/arm64/Kconfig.platforms`, commit `ecf6405fa6c5`) and placed Tensor G5 device trees directly under `arch/arm64/boot/dts/google/` (`lga.dtsi`, `lga-mustang.dts`, commit `cf3df78629f2`) rather than `arch/arm64/boot/dts/exynos/google/` `[V]` `U/arch-google`, `U/lga-dts`. `device.conf` therefore sets `DTB=google/lga-mustang.dtb` and `device/mustang/kernel.config` sets `CONFIG_ARCH_GOOGLE=y`.
2. **No `/soc_id` and no `/ect`:** Unlike `gs101`/`gs201`, `lga.dtsi` has neither `/soc_id` nor an Exynos `/ect` table `[V]` `U/lga-dts`. Instead, ABL matches `/soc_compatible` children (`product_id`, `major`, `minor`, `pkg_mode`) `[V]` `U/lga-dts`:
   - `A0`: `product_id = <0x5>; major = <0x0>; minor = <0x0>; pkg_mode = <0x0>;` (`0x000500`) `[V]`
   - `B0`: `product_id = <0x5>; major = <0x1>; minor = <0x0>; pkg_mode = <0x0>;` (`0x000510`, retail silicon) `[V]`
   This matches `SOC_COMPATIBLE="b0:0x5:1:0:0 a0:0x5:0:0:0"` in open `bin/omamobile` PR `#27`. Per `L-44`, `SOC_IDS` and `SOC_COMPATIBLE` remain commented out in `device.conf` so both `bin/omamobile` and `tools/install-phone` fail closed until `lga-mustang.dtb` is on `linux-omarchy-mobile`, PR `#27` merges, and stock boot image headers are confirmed on hardware (`[I]` whether stock puts `dtb.img` in `vendor_boot` or `vendor_kernel_boot`).
3. **Mandatory `/aliases/ufs0` node in FDT (`overlays/ufs-placeholder.dtso`):** Stock Laguna ABL reads `/aliases/ufs0` during FDT fixup and aborts boot if the alias or its target node is missing `[V]` `U/lga-dts`, `S/pixelscripts`. Until `ufs@3c400000` is enabled in `lga.dtsi`, `DTB_OVERLAYS="ufs-placeholder"` applies [`overlays/ufs-placeholder.dtso`](overlays/ufs-placeholder.dtso) (`target-path = "/"`, works with `fdtoverlay` even when the base DTB is compiled without `-@`).
4. **Hardware watchdog:** Stock ABL leaves the hardware watchdog armed across handoff; before booting a mainline kernel without the vendor watchdog pet driver, an unlocked bootloader **must** run `fastboot oem watchdog disable` or the phone resets roughly 15–30 seconds into boot `[V]` `S/pixelscripts`.
5. **Warm reset before PMU/CPM reboot-mode lands:** Until `goog_cpm_mailbox` / reboot-mode lands in the kernel tree, `echo b > /proc/sysrq-trigger` triggers a clean warm reset (hold Volume Down during reset to enter `fastboot`) `[V]` `S/pixelscripts`.

## Partition layout and storage guard

Sizes from `A/muzel` (`BoardConfig-common.mk`) `[V]`:

| Partition | Size | Header / notes | Guard disposition (`initramfs/init`) |
| --- | --- | --- | --- |
| `boot_a` / `boot_b` | `67,108,864 B` (`64 MiB`, `0x4000000`) | Boot image v4 (`Image.lz4`); chained in AVB footer | `boot_a` **RO**, `boot_b` **RW** (`UFS_RW`) |
| `init_boot_a` / `init_boot_b` | `8,388,608 B` (`8 MiB`, `0x800000`) | Boot image v4 (empty generic ramdisk, OS version `17.0.0` / `2026-10`); chained in AVB footer | `init_boot_a` **RO**, `init_boot_b` **RW** (`UFS_RW`) |
| `vendor_boot_a` / `vendor_boot_b` | `67,108,864 B` (`64 MiB`, `0x4000000`) | Boot image v4 (`pagesize = 0x1000`, cmdline, bootconfig) | `vendor_boot_a` **RO**, `vendor_boot_b` **RW** (`UFS_RW`) |
| `vendor_kernel_boot_a` / `vendor_kernel_boot_b` | `67,108,864 B` (`64 MiB`, `0x4000000`) | Boot image v4 (`dtb.img` + `initramfs.cpio.lz4`, `pagesize = 0x1000`; verify live header in M1 step 1) | `vendor_kernel_boot_a` **RO**, `vendor_kernel_boot_b` **RW** (`UFS_RW`) |
| `dtbo_a` / `dtbo_b` | `16,777,216 B` (`16 MiB`, `0x1000000`) | `mkdtboimg.py` (`page_size = 4096`, `id = 0x70506`, `rev = 0x10000`) | `dtbo_a` **RO**, `dtbo_b` **RW** (`UFS_RW`) |
| `pvmfw_a` / `pvmfw_b` | `1,048,576 B` (`1 MiB`) | Protected VM firmware | **RO** (never written) |
| `vbmeta_a` / `vbmeta_b` / `vbmeta_system_*` / `vbmeta_vendor_*` | VBMeta descriptors | Do **not** overwrite or `--disable-verity`; unlocked ABL accepts AOSP test-key hash footers on `boot_b`/`init_boot_b` | **RO** (never written) |
| `super` | `8,531,214,336 B` (`google_dynamic_partitions_size = 8,527,020,032 B`) | Shared slot A/B dynamic partitions (`system`, `system_dlkm`, `system_ext`, `product`, `vendor`, `vendor_dlkm`) — **never touch** | **RO** (never written) |
| `userdata` | Remaining LU0 capacity | `omamobile-root` ext4 filesystem (`zufs_provisioned=true` on stock Android; reformatted ext4 by `install-phone`) | **RW** (`UFS_RW`) |
| `persist`, `efs`, `efs_backup`, `modem_*`, `bootloader_*`, `devinfo`, `misc`, `bl1_*` (`sdb`/`sdc`) | Calibration, IMEI, bootloader LUNs, slot metadata | Hardware calibration and boot firmware | **RO** (`blockdev --setro` enforced in PID 1; `/dev/sda` whole-disk node unlinked) |

## Subsystem and MMIO hardware map

Every physical base address and driver name below is verified `[V]` from `U/*` and `A/laguna` (`vendor.prop`, `sepolicy/vendor/genfs_contexts`, `proprietary-files-vendor.txt`):

| Subsystem | Laguna hardware / MMIO | Stock vendor driver & firmware | Mainline / `omarchy-mobile` status |
| --- | --- | --- | --- |
| Debug UART (`ttyS0`) | `serial@db62000` (`0x0db62000`), `osc` `38.4 MHz`, `"google,lga-uart", "snps,dw-apb-uart"`; peripheral UARTs `3a352000.uart` (`ttyS1`), `3a322000.uart` (`ttyS2`) | `google_8250` / `8250_dw` | **In mainline Linux v7.4** (`b6932e768405`, `CONFIG_SERIAL_8250_DW=y`, `# CONFIG_ACPI is not set`). **Never** use Exynos `earlycon=exynos4210,0x10A00000` `[V]` `U/uart-binding`, `S/pixelscripts` |
| USB 3.2 Gen 2 & UDC | Wrapper `usb@c400000` (`0x0c400000`), core `c450000.usb3`, PHY `usb-phy@c410000` (`0x0c410000`) | `dwc3-google`, `phy-google-usb`, `vendor/firmware/usb_phy_fw_2.27.0.bin` | **In mainline Linux v7.4** (`18686b294c6f`, `42d6a93bff56`, `54edd8698bb1`; `CONFIG_USB_DWC3_GOOGLE=y`, `CONFIG_PHY_GOOGLE_USB=y`, `CONFIG_TYPEC=y`). Primary M1 SSH path (`172.16.42.1`) `[V]` `U/dwc3-google`, `U/phy-google` |
| USB Type-C / PD | `max77759tcpc` (`0-04` on SPMI `53f1000.spmi`) | `tcpci_max77759` | Bootloader establishes initial sink/device role for tethered fastboot->Linux boot `[I]`; full dynamic role switch needs SPMI + `max77759` TCPC bring-up `[V]` `A/laguna` |
| Display (DPU + DSI-2) | VeriSilicon DC9x00 DPU `sswrp_dpu@ec00000` (`0x0ec00000`), primary Synopsys MIPI DSI-2 host `efd8000.dsi2host`, secondary `eff8000.dsi2host` | `vs_drm` + `mipi_dsi2h` (not Samsung DECON/DSIM) | M1 uses `CONFIG_DRM_SIMPLEDRM=y` over ABL's pre-lit framebuffer. **Leave `SIMPLEFB_TRIG_CON` and `BRINGUP_PROBE` unset** (`L-4`). Upstream has early `drivers/gpu/drm/verisilicon/` and `dw-mipi-dsi2.c` building blocks `[V]` `A/laguna`, `U/vs-drm` |
| GPU | Imagination PowerVR `DXT-48-1536` at `34f00000.gpu0` (`0x34f00000`, Volcanic architecture, BVNC `71.2.2448.1212`) | `pvrsrvkm`, signed fw `rgx.fw.signed.71.2.2448.1212`, shader bin `rgx.sh.71.2.2448.1212` | Mainline `drm/imagination` (`pvr_drv.c`) currently binds only Rogue cores (`33.15.11.3`, `36.52.104.182`, `36.53.104.796`), **not** Volcanic `71.2.2448.1212` `[V]` `U/pvr`. M1 desktop runs software-rendered via `llvmpipe` + `packages/hyprland` (`software-renderer.patch`) `[V]` |
| Storage (UFS) | `3c400000.ufs` (`0x3c400000`) | Google Laguna `ufshcd` glue | Port `ufs@3c400000` + clocks/PHY from Google GPL kernel into `linux-omarchy-mobile` before M1 `userdata` switch-root (M1 steps 4–5 verify RAM initramfs first; step 6 requires UFS LU0 enumeration) `[V]` `U/lga-dts`, `A/laguna` |
| Interrupts, pinctrl, clocks, mailbox | `irq_gia_google`, `pinctrl_google`, `goog_cpm_mailbox` / `goog_nq_mailbox`, `i2c_google`, `spi_dw_mmio`, `spmi_smartdv` | Google Laguna SoC drivers | Not yet in `torvalds/linux.git` beyond base `lga.dtsi` GICv3 + `osc`; port required blocks (`pinctrl`, `irq-gia`, `cpm`, `ufs`, `spmi`, `i2c`) cleanly from Google's GPL kernel `[V]` `A/laguna` |
| Crash log (`pstore`) | `ramoops` reserved memory at `0x95200000`, size `0x400000` (`4 MiB`) | `pstore` / `ramoops` | Supported by `CONFIG_PSTORE_RAM=y` already enabled in `kernel/configs/omamobile.config` once reserved-memory node matches live DT `[V]` `A/laguna` |
| Touchscreen | FocalTech `ft3683u` (`focal_touch.ko`, firmware `ft3683u_firmware.bin`) | `focal_touch` (replaces `cheetah`'s Synaptics S3908) | Port SPI + `pinctrl_google` + `ft3683u` input driver in M2 after M1 SSH + display `[V]` `A/laguna` |
| Power / PMIC / Charger / FG | SPMI controllers `53f1000.spmi` and `ebf1000.spmi`; `max77779` PMIC/FG/chg (`0-03`, `0-06`), `ln8411` switched-cap charger (`44-006e`) | `spmi_smartdv`, `regmap-goog-spmi`, `max77779_*`, `ln8411` | Post-M1 milestone (`M3`); keep phone above 50% SoC during M1 tests until charger driver is active `[V]` `A/laguna` |
| PCIe & Wi-Fi / BT / Thread | PCIe RC `3c500000.pcie` (`laguna_pcie_phy_fw.bin`) + Broadcom `BCM4390` (`fw_bcmdhd.bin_4390_b1`, `BCM4390B1_*.hcd`) | `pcie-google`, `bcmdhd4390` / `brcmfmac` | Post-M1 milestone; evaluate upstream `brcmfmac` BCM4390 PCIe IDs vs clean port `[V]` `A/laguna` |
| Audio & Haptics | Google AoC `9000000.aoc` (`aoc.bin`) + Cirrus `CS35L43` speaker amps (`cs35l43-dsp1-spk-prot.{bin,wmfw}`) + Cirrus `CS40L26` haptics (`10-0043` on `3a331000.i2c`, `cs40l26.bin`) | `aoc`, `snd_soc_cs35l43`, `cs40l26` | Mainline Linux already ships `snd-soc-cs35l43` and `input/misc/cs40l26`; AoC transport requires Laguna mailbox/audio port `[V]` `A/laguna` |
| Cellular modem | Samsung Exynos Modem 5400 (`modem_a`/`modem_b`, SIT RIL) | `cpif` + SIT RIL (`omamobile-modem-sit` / `omamobile-mmd`) | Keep modem powered down and `efs`/`persist`/`modem_*` read-only through bring-up `[V]` `A/laguna` |
| Fingerprint | Qualcomm QFS ultrasonic under-display sensor (`qfs.app` TZ app) | `qbt_handler` + `omamobile-fingerprint` | Post-desktop milestone `[V]` `A/laguna` |

## Firmware policy

Per [`CONTRIBUTING.md`](../../CONTRIBUTING.md) §1 and §7, **no proprietary blob is ever copied into this repository or `linux-omarchy-mobile`**. On hardware, vendor firmware files (such as `usb_phy_fw_2.27.0.bin`, `laguna_pcie_phy_fw.bin`, `ft3683u_firmware.bin`, `cs35l43-dsp1-spk-prot.*`, `cs40l26.bin`, and `rgx.fw.signed.71.2.2448.1212`) are read or extracted locally from the phone's own stock `vendor` partition or Google's official factory archive at install/provision time (`tools/phone-provision`).

## Host setup

Same USB CDC-NCM gadget address plan as every Omarchy Mobile device:

- Phone `usb0`: `172.16.42.1/24` (initramfs Dropbear SSH and systemd-networkd [`20-usb-gadget.network`](../../packages/omamobile-device-mustang/vendor/systemd/network/20-usb-gadget.network))
- Host USB interface: `172.16.42.2/24` (NetworkManager `Shared to other computers` or manual `172.16.42.2/24` with NAT)

## Sources

| ID | License | Citation |
| --- | --- | --- |
| `U/lga-soc` | GPL-2.0 / BSD-2-Clause | Upstream Linux v7.4 commit `54edd8698bb1` (*dt-bindings: arm: google: Document Google Tensor G5 (Laguna) SoC and Pixel 10 boards*) — `Documentation/devicetree/bindings/arm/google.yaml` |
| `U/arch-google` | GPL-2.0-only | Upstream Linux v7.4 commit `ecf6405fa6c5` (*arm64: Kconfig: Add CONFIG_ARCH_GOOGLE for Google Tensor SoCs*) — `arch/arm64/Kconfig.platforms` |
| `U/lga-dts` | GPL-2.0-only | Upstream Linux v7.4 commit `cf3df78629f2` (*arm64: dts: google: Add initial Google Tensor G5 (Laguna) and Pixel 10 series DTs*) and LKML series `20251111192422.4180216-1-dianders@chromium.org` (`arch/arm64/boot/dts/google/lga.dtsi`, `lga-mustang.dts`) |
| `U/uart-binding` | GPL-2.0 / BSD-2-Clause | Upstream Linux v7.4 commit `b6932e768405` (*dt-bindings: serial: snps-dw-apb-uart: Add "google,lga-uart"*) |
| `U/dwc3-google` | GPL-2.0-only | Upstream Linux v7.4 commit `18686b294c6f` (*usb: dwc3: Add Google Tensor G5 (Laguna) DWC3 glue driver*) — `drivers/usb/dwc3/dwc3-google.c`, `Documentation/devicetree/bindings/usb/google,lga-dwc3.yaml` |
| `U/phy-google` | GPL-2.0-only | Upstream Linux v7.4 commit `42d6a93bff56` (*phy: Add Google Tensor G5 USB PHY driver*) — `drivers/phy/phy-google-usb.c`, `Documentation/devicetree/bindings/phy/google,lga-usb-phy.yaml` |
| `U/pvr` | GPL-2.0-only / MIT | Upstream Linux `drivers/gpu/drm/imagination/pvr_drv.c` and `Documentation/gpu/imagination/index.rst` |
| `U/vs-drm` | GPL-2.0-only | Upstream Linux `drivers/gpu/drm/verisilicon/` and `drivers/gpu/drm/bridge/synopsys/dw-mipi-dsi2.c` |
| `S/pixelscripts` | Apache-2.0 / GPL-2.0 | Linaro Google Landing Team bring-up scripts & overlays: `https://gitlab.com/LinaroLtd/googlelt/pixelscripts` (`lga-ufs-placeholder.dtso`, `fastboot oem watchdog disable`, `ttyS0` / `CONFIG_ACPI=n`, `fastboot stage vendor_boot.img && fastboot boot boot.img`) |
| `F/images` | Public fact table (no blobs copied) | Google Factory Images for Nexus and Pixel Devices (`mustang`, `CP3A.261005.005`, Oct 2026): `https://developers.google.com/android/images` |
| `F/ota` | Public fact table (no blobs copied) | Google Full OTA Images for Nexus and Pixel Devices (`mustang`, `CP3A.261005.005`, Oct 2026): `https://developers.google.com/android/ota` |
| `A/muzel` | Apache-2.0 | `https://github.com/LineageOS/android_device_google_muzel` (`lineage-24.0`, `BoardConfig-common.mk`) |
| `A/mustang` | Apache-2.0 | `https://github.com/LineageOS/android_device_google_mustang` (`lineage-24.0`, `init_mustang.cpp`, device properties) |
| `A/laguna` | Apache-2.0 | `https://github.com/LineageOS/android_device_google_laguna` (`lineage-24.0`, `BoardConfig-common.mk`, `vendor.prop`, `sepolicy/vendor/genfs_contexts`, `proprietary-files-vendor.txt`) |
