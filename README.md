# BL32 (OP-TEE) port for RK3576 devices

Patches and artifacts to enable OP-TEE (BL32) on Rockchip RK3576 devices
with mainline TF-A + OP-TEE OS + U-Boot.

Tested on **Radxa Rock 4D** (linux-next 7.1.0-rc5-next-20260527,
`CONFIG_OPTEE=y`). Firmware lives on SPI flash; kernel + xtest initramfs
boot from SD card.

xtest result: **113 tests, 1 failed** (regression_1033 — plugin TA,
`CONFIG_TEE_SUPP_PLUGIN` missing from initramfs build, not a platform
issue).

OP-TEE work-in-progress branch:
[gahingwoo/optee_os — local-rk3576-test](https://github.com/gahingwoo/optee_os/tree/local-rk3576-test)

Thanks to [the-gabe](https://github.com/the-gabe) for the RK3576 OTP
layout analysis (HUK index 0x80, RSA hash index 0x184) that informed
patches 0003 and 0006.

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

All patches apply to **OP-TEE OS** on top of commit `ccb894f`.

```bash
cd optee_os
git apply ../0001-plat-rockchip-add-RK3576-platform-support.patch
git apply ../0002-plat-rockchip-rk3576-switch-debug-UART-to-UART0-and-.patch
git apply ../0003-plat-rockchip-rk3576-add-OTP-HUK-derivation-via-Secu.patch
git apply ../0004-plat-rockchip-rk3576-add-RKRNG_S-hardware-TRNG-drive.patch
git apply ../0005-plat-rockchip-rk3576-force-CFG_CRYPTO_WITH_CE-y.patch
git apply ../0006-plat-rockchip-rk3576-add-secure-boot-PTA-support-RSA.patch
```

### `0001` — Initial RK3576 platform support *(upstream PR #7821)*

Adds `PLATFORM_FLAVOR=rk3576` to `plat-rockchip`:
- `conf.mk`: 8 cores (4×A72 + 4×A53), GICv2, TZDRAM `0x70000000+32 MiB`,
  SHMEM `0x72000000+4 MiB`
- `platform_config.h`: GIC, UARTs, SGRF/Firewall addresses
- `platform_rk3576.c`: `platform_secure_ddr_region()` via SYS_SGRF_FW
- `sub.mk`: hook into build

### `0002` — UART0 early console

TF-A uses UART0 (`0x2ad40000`) as its debug console; OP-TEE defaulted to
UART2. Forces `CFG_EARLY_CONSOLE=y` — TF-A does not pass a DT pointer to
BL32 so the DT-based console probe never runs.

### `0003` — OTP HUK derivation *(Co-authored: the-gabe)*

Implements `tee_otp_get_hw_unique_key()` via the shared `rockchip_otp.c`
driver. Reads the OTP slot first; falls back to an ephemeral SW-PRNG key
on unprogrammed boards.

- `ROCKCHIP_OTP_HUK_INDEX = 0x80` (OTP_S words 0x80–0x83, bytes 512–527)
- Differs from RK3588 (`0x104`)
- OTP writes (provisioning) gated by `CFG_RK3576_PERSIST_HUK=n` (default off — irreversible)

### `0004` — RKRNG_S hardware TRNG driver

RK3576 uses the RKRNG IP (`0x2a440000`) rather than RK3588's TRNG_V1.
Provides `hw_get_random_bytes()` and overrides `plat_get_random_stack_canaries()`
so `CFG_WITH_SOFTWARE_PRNG=n` works cleanly. Enabling `CFG_RK3576_RKRNG=y`
automatically sets `CFG_WITH_SOFTWARE_PRNG=n`.

Two ordering issues solved:
1. `hw_get_random_bytes()` lazily maps RKRNG on first call (safe because
   `core_init_mmu_map` runs in `entry_a64.S` before any initcall)
2. `plat_get_random_stack_canaries()` override reads RKRNG directly before
   `driver_init()` fires

### `0005` — Force `CFG_CRYPTO_WITH_CE=y`

Cortex-A55 and A72 both implement ARMv8 Cryptographic Extensions.
Matches RK3588 behaviour.

### `0006` — Secure Boot PTA (RSA-2048) *(Co-authored: the-gabe)*

Enables `CFG_RK_SECURE_BOOT` with the RK3576 OTP layout:

- Status word: OTP_S word `0x8` (same as RK3588)
- RSA key hash: OTP_S words `0x184`–`0x187` (SHA-256 of RSA-2048 public key)
- RK3576 supports RSA-2048 only (no RSA-4096 status fuse)

`CFG_RK_SECURE_BOOT_SIMULATION=y` by default — set to `n` only when ready
to permanently fuse the hash (irreversible, may brick device).

Also fixes `rk_secure_boot.c` to support variable hash sizes:
removes `static_assert(size == 8)`, makes `otp_to_string()` variable-length,
gates the RSA-4096 path on `#ifdef ROCKCHIP_OTP_SECURE_BOOT_STATUS_RSA4096`.

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
# 1. OP-TEE (with RKRNG hardware entropy)
cd optee_os
make PLATFORM=rockchip-rk3576 \
     CROSS_COMPILE=aarch64-linux-gnu- \
     CROSS_COMPILE_core=aarch64-linux-gnu- \
     CROSS_COMPILE_ta_arm64=aarch64-linux-gnu- \
     CFG_ARM64_core=y CFG_USER_TA_TARGETS=ta_arm64 \
     CFG_RK3576_RKRNG=y \
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

```bash
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

Key cmdline notes:
- `kernel_addr_r=0x50000000` — load kernel above SHMEM
- `nokaslr` — linux-next has `CONFIG_RANDOMIZE_BASE=y`; KASLR causes a silent hang
- `clk_ignore_unused` — prevents the clock framework from gating UART0 early

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

```bash
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
