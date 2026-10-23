# omamobile-device-mustang

Omarchy Mobile's initial userspace package for the Google Pixel 10 Pro XL (codename `mustang`, Tensor G5 / `laguna`). It hides Android UFS partitions from desktop tools, holds the USB CDC-NCM gadget address after `switch_root`, and configures pacman's `[omamobile]` repository. It does not mark a boot slot successful or start radio, charger, or camera services before each subsystem has its own hardware runbook.

| Path | Installs to | What it does |
| --- | --- | --- |
| `vendor/udev/rules.d/10-omamobile-storage.rules` | `/usr/lib/udev/rules.d` | Keeps `/dev/sda` mode `0000` and hides every Android partition except `userdata` from desktop tools (`UDISKS_IGNORE=1`). Same rule as `omamobile-device-cheetah` |
| `vendor/systemd/network/20-usb-gadget.network` | `/usr/lib/systemd/network` | Assigns `172.16.42.1/24` on `usb0` after handover |
| `vendor/NetworkManager/conf.d/10-omamobile-usb.conf` | `/usr/lib/NetworkManager/conf.d` | Keeps NetworkManager off `usb0` |
| `etc/omamobile-mirrorlist` | `/etc/pacman.d/omamobile-mirrorlist` | Points pacman at `/var/lib/omamobile/repo/$arch`. Same file as `omamobile-device-cheetah` |
| `omamobile-device-mustang.install` | pacman hook | Inserts `[omamobile]` before `[omarchy]` in `/etc/pacman.conf` once. Same script as `omamobile-device-cheetah` |
| `src/reboot-to.c` | `/usr/bin/reboot-to` | Invokes `SYS_reboot(..., LINUX_REBOOT_CMD_RESTART2, reason)` (`bootloader` by default). Note: until Laguna reboot-mode support lands in the kernel tree, use `echo b > /proc/sysrq-trigger` while holding Volume Down as fallback (`device/mustang/BRINGUP-M1.md`) |
| `device.conf` | `/usr/share/omamobile/device.conf` | Byte-identical copy of `device/mustang/device.conf`, verified by `test/all` |

`./build [DIR]` compiles `reboot-to` inside this directory, `./install DESTDIR [DIR]` stages all files, and `test/all` verifies zero drift against `packages/omamobile-device-cheetah` and `device/mustang/device.conf`.

Not in this package: `omamobile-boot-success`, `omamobile-bootctl`, Wi-Fi, charger guard, and cameras. Each enters only after its dedicated runbook passes on hardware.
