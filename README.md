# BL32 (OP-TEE) port for RK3576 devices

Patches and artifacts to enable OP-TEE (BL32) on Rockchip RK3576 devices
with mainline TF-A + OP-TEE OS + U-Boot.

Tested on **Radxa Rock 4D** (linux-next 7.1.0-rc5-next-20260527,
`CONFIG_OPTEE=y`). Firmware lives on SPI flash; kernel + xtest initramfs
boot from SD card.

xtest result: **113 tests, 1 failed** (regression_1033 — plugin TA,
`CONFIG_TEE_SUPP_PLUGIN` missing from initramfs build, not a platform
issue).

---

## Component versions

| Component | Version / commit |
|-----------|-----------------|
| TF-A      | v2.14.0 (`2a313ed`) |
| OP-TEE OS | 4.10 (`ccb894f`) |
| U-Boot    | v2026.04-rc1 |
| Kernel    | linux-next 7.1.0-rc5-next-20260527 (`CONFIG_OPTEE=y`) |
| DDR blob  | `rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin` (rkbin) |

---

## Patches

### `0001-plat-rockchip-add-RK3576-platform-support.patch`
Apply to **OP-TEE OS**.

Adds `PLATFORM_FLAVOR=rk3576` to `plat-rockchip`, covering:
- `conf.mk`: 8 cores (4×A72 + 4×A53), GIC-400 (GICv2), TZDRAM at
  `0x70000000..+32 MiB`, SHMEM at `0x72000000..+4 MiB`.
- `platform_config.h`: GIC, UARTs, SGRF/Firewall addresses.
- `platform_rk3576.c`: `platform_secure_ddr_region()` via SYS_SGRF_FW.
- `sub.mk`: hook into build.

```bash
cd optee_os
git apply ../0001-plat-rockchip-add-RK3576-platform-support.patch
```

### `0002-tfa-rk3576-fix-GICV2_G0_FOR_EL3-for-SPD-opteed.patch`
Apply to **TF-A** (`plat/rockchip/rk3576/platform.mk`).

RK3576 TF-A hardcoded `GICV2_G0_FOR_EL3 := 1`, routing all Group-0
secure interrupts to EL3. With `SPD=opteed` this causes
`register_interrupt_type_handler()` to return `-EINVAL` and
`opteed_main.c` to `panic()` immediately after OP-TEE returns from
init → reset loop.

Fix: make `GICV2_G0_FOR_EL3` conditional on `SPD`.

```bash
cd tfa
git apply ../0002-tfa-rk3576-fix-GICV2_G0_FOR_EL3-for-SPD-opteed.patch
```

### `0003-optee-rk3576-switch-debug-uart-to-uart0-force-early-console.patch`
Apply to **OP-TEE OS**.

TF-A RK3576 uses UART0 @ `0x2ad40000` as its debug console. OP-TEE
defaulted to UART2. TF-A does not pass a DT pointer to BL32, so
`CFG_EARLY_CONSOLE` must be forced on.

```bash
cd optee_os
git apply ../0003-optee-rk3576-switch-debug-uart-to-uart0-force-early-console.patch
```

### `0004-optee-rk3576-add-otp-huk-derivation.patch`
Apply to **OP-TEE OS**.

Implements `tee_otp_get_hw_unique_key()` for RK3576 via the shared
`rockchip_otp.c` driver. Reads OTP slot first; falls back to ephemeral
SW-PRNG key on unprogrammed boards.

> **Note:** `ROCKCHIP_OTP_HUK_INDEX = 0x80` (OTP_S words 0x80–0x83,
> bytes 512–527). This differs from RK3588 (0x104). RK3576 OTP_S has
> 512 words (0x200 max). OTP writes are irreversible — set
> `CFG_RK3576_PERSIST_HUK=y` only when ready to commit the HUK.

```bash
cd optee_os
git apply ../0004-optee-rk3576-add-otp-huk-derivation.patch
```

### `0005-optee-rk3576-explicit-sw-prng.patch`
Apply to **OP-TEE OS**.

Forces `CFG_WITH_SOFTWARE_PRNG=y`. The Secure TRNG at `0x2a440000`
does not respond on the Radxa Rock 4D; this patch ensures a working
PRNG until the TRNG address is confirmed from the RK3576 TRM.

```bash
cd optee_os
git apply ../0005-optee-rk3576-explicit-sw-prng.patch
```

---

## Memory map

```
0x40000000  TF-A BL31 TZRAM
0x40800000  U-Boot
0x70000000  OP-TEE TZDRAM (32 MiB, secure, DDR firewall + no-map)
0x72000000  OP-TEE shared memory (4 MiB, non-secure)
```

---

## Build

```bash
# 1. OP-TEE
cd optee_os
make PLATFORM=rockchip-rk3576 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     CROSS_COMPILE_core=aarch64-linux-gnu- \
     CROSS_COMPILE_ta_arm64=aarch64-linux-gnu- \
     CFG_ARM64_core=y CFG_USER_TA_TARGETS=ta_arm64 \
     DEBUG=1 CFG_TEE_LOGLEVEL=3 -j$(nproc)
cp out/arm-plat-rockchip/core/tee.bin ../out/tee.bin

# 2. TF-A BL31
cd ../tfa
make PLAT=rk3576 ARCH=aarch64 CROSS_COMPILE=aarch64-linux-gnu- \
     SPD=opteed BL32=../out/tee.bin DEBUG=1 LOG_LEVEL=40 -j$(nproc)
cp build/rk3576/debug/bl31/bl31.elf ../out/bl31.elf

# 3. U-Boot (v2026.04-rc1; v2026.07-rc2 has a broken atf-3 hash on RK3576)
cd ../build/u-boot
export CROSS_COMPILE=aarch64-linux-gnu-
export BL31=../../out/bl31.elf
export TEE=../../out/tee.bin
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin
make rock-4d-rk3576_defconfig
make -j$(nproc)
cp u-boot-rockchip-spi.bin ../../out/
cp u-boot-rockchip.bin     ../../out/
```

---

## Flash firmware (SPI via MASKROM)

```bash
rkdeveloptool ld
rkdeveloptool db build/rkbin/bin/rk35/rk3576_usbplug_v1.04.bin
rkdeveloptool ef
rkdeveloptool wl 0 out/u-boot-rockchip-spi.bin
rkdeveloptool rd
```

---

## xtest SD card image

Build a self-contained SD card image (firmware on SPI, kernel + xtest
initramfs on SD).

### 1. Build xtest

```bash
# OP-TEE test suite
cd /path/to/optee_test
make \
  CROSS_COMPILE=aarch64-linux-gnu- \
  TA_DEV_KIT_DIR=/path/to/optee_os/out/arm-plat-rockchip/export-ta_arm64 \
  OPTEE_CLIENT_EXPORT=/path/to/optee_client/out/export/usr \
  CFG_USER_TA_TARGETS=ta_arm64 \
  O=out -j$(nproc)
```

### 2. Pack initramfs

Assemble a minimal initramfs (busybox + tee-supplicant + libteec + xtest
+ TAs) and pack it as `initramfs.cpio.gz`.

### 3. Prepare merged DTB

The kernel DTB must reserve OP-TEE memory so the kernel does not map the
DDR-firewall-protected region:

```bash
# Update out/rk3576-optee.dts addresses to match build (0x70000000/0x72000000)
dtc -@ -I dts -O dtb -o out/rk3576-optee.dtbo out/rk3576-optee.dts
fdtoverlay \
  -i /path/to/kernel/arch/arm64/boot/dts/rockchip/rk3576-rock-4d.dtb \
  -o out/rk3576-rock-4d-optee.dtb \
  out/rk3576-optee.dtbo
```

### 4. Build SD image

```bash
# boot.cmd (compile to boot.scr with mkimage):
setenv kernel_addr_r  0x50000000
setenv fdt_addr_r     0x5f000000
setenv ramdisk_addr_r 0x60000000
setenv bootargs "console=ttyS0,1500000n8 earlycon=uart8250,mmio32,0x2ad40000 nokaslr rdinit=/init clk_ignore_unused"
load mmc ${devnum}:1 ${kernel_addr_r}  Image
load mmc ${devnum}:1 ${fdt_addr_r}     rk3576-rock-4d.dtb
load mmc ${devnum}:1 ${ramdisk_addr_r} initramfs.cpio.gz
setenv initrd_size ${filesize}
booti ${kernel_addr_r} ${ramdisk_addr_r}:${initrd_size} ${fdt_addr_r}
```

Key kernel cmdline notes:
- `kernel_addr_r=0x50000000` — load kernel above SHMEM; do not use the
  U-Boot default `0x42000000` (conflicts with earlier TZDRAM layouts).
- `nokaslr` — linux-next has `CONFIG_RANDOMIZE_BASE=y`; KASLR causes a
  silent hang on this hardware.
- `clk_ignore_unused` — prevents the RK3576 clock framework from gating
  UART0 before the serial driver initializes.

Layout (MBR, no-root — initramfs only):
```
sector 64        U-Boot (out/u-boot-rockchip.bin)
16 MiB → end     FAT32: boot.scr, Image, rk3576-rock-4d.dtb, initramfs.cpio.gz
```

Flash:
```bash
sudo dd if=optee-xtest-rock4d.img of=/dev/sdX bs=4M status=progress conv=fsync
```

Serial console: `ttyS0` @ 1500000 n8 (same cable as TF-A / U-Boot).

---

## Linux DT overlay (Armbian / distro)

For running OP-TEE on a distro kernel without rebuilding:

```bash
# Compile overlay (addresses must match firmware build)
dtc -@ -I dts -O dtb -o rk3576-optee.dtbo out/rk3576-optee.dts

# Install (Armbian)
sudo install -m 0644 rk3576-optee.dtbo /boot/dtb/rockchip/overlay/
grep -q '^overlays=' /etc/armbianEnv.txt \
  && sudo sed -i '/^overlays=/{/rk3576-optee/!s/$/ rk3576-optee/}' /etc/armbianEnv.txt \
  || echo 'overlays=rk3576-optee' | sudo tee -a /etc/armbianEnv.txt
sudo reboot
```

After reboot:
```bash
dmesg | grep -i optee
sudo apt install -y optee-client && sudo systemctl enable --now tee-supplicant
```
