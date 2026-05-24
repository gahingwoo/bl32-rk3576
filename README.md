# BL32 (OP-TEE) port for RK3576 devices

Patches and artifacts to enable OP-TEE (BL32) on Rockchip RK3576 devices
with mainline TF-A + OP-TEE OS + U-Boot.

Tested on **Radxa Rock 4D** (Armbian 26.2, kernel 7.0.6-edge-rockchip64).

---

## Component versions

| Component | Version / commit |
|-----------|-----------------|
| TF-A      | v2.14.0 (`dc7c828`) |
| OP-TEE OS | `d820f13-dev` |
| U-Boot    | v2026.04-rc1 |
| DDR blob  | `rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin` (rkbin) |

---

## Patches

### `0001-plat-rockchip-add-RK3576-platform-support.patch`
Apply to **OP-TEE OS**.

Adds `PLATFORM_FLAVOR=rk3576` to `plat-rockchip`, covering:
- `conf.mk`: 8 cores (4×A72 + 4×A53), GIC-400 (GICv2), TZDRAM at
  `0x40400000..+32 MiB`, SHMEM at `0x42400000..+4 MiB`.
- `platform_config.h`: GIC, UARTs, SGRF/Firewall, CRU, SRAM addresses.
- `platform_rk3576.c`: `platform_secure_ddr_region()` via System SGRF Firewall.
- `sub.mk`: hook into build.

```bash
cd optee_os
git apply ../0001-plat-rockchip-add-RK3576-platform-support.patch
```

### `0002-tfa-rk3576-fix-GICV2_G0_FOR_EL3-for-SPD-opteed.patch`
Apply to **TF-A** (`plat/rockchip/rk3576/platform.mk`).

RK3576 TF-A hardcoded `GICV2_G0_FOR_EL3 := 1`, which routes all Group-0
secure interrupts to EL3. With `SPD=opteed` this causes
`plat_ic_has_interrupt_type(INTR_TYPE_S_EL1)` to return false, so
`register_interrupt_type_handler()` returns `-EINVAL` and `opteed_main.c`
calls `panic()` immediately after OP-TEE returns from init → reset loop.

Fix: make `GICV2_G0_FOR_EL3` conditional on `SPD`.

```bash
cd tfa
git apply ../0002-tfa-rk3576-fix-GICV2_G0_FOR_EL3-for-SPD-opteed.patch
```

### `0003-optee-rk3576-switch-debug-uart-to-uart0-force-early-console.patch`
Apply to **OP-TEE OS** (`core/arch/arm/plat-rockchip/conf.mk`).

TF-A RK3576 uses UART0 @ `0x2ad40000` as its debug console. OP-TEE
defaulted to UART2. Also, TF-A does not pass a DT pointer to BL32, so
`CFG_EARLY_CONSOLE` must be forced on — otherwise OP-TEE has no console.

```bash
cd optee_os
git apply ../0003-optee-rk3576-switch-debug-uart-to-uart0-force-early-console.patch
```

---

## Memory map

```
0x40040000  BL31 (TZRAM, ~136 KiB)
0x40400000  OP-TEE TZDRAM (32 MiB, secure, no-map)
0x42400000  OP-TEE shared memory (4 MiB, non-secure)
0x40800000  U-Boot
```

---

## Build

```bash
# 1. TF-A BL31 — must unset BL31 env var or make skips the build
unset BL31 TEE ROCKCHIP_TPL
make -C tfa CROSS_COMPILE=aarch64-linux-gnu- PLAT=rk3576 SPD=opteed DEBUG=1 -j$(nproc) bl31

# 2. OP-TEE
make -C optee_os CROSS_COMPILE64=aarch64-linux-gnu- PLATFORM=rockchip-rk3576 \
     CFG_ARM64_core=y CFG_TEE_CORE_LOG_LEVEL=2 -j$(nproc)

# 3. U-Boot (use v2026.04-rc1; v2026.07-rc2 has a broken atf-3 hash on RK3576)
cd build/u-boot
git checkout v2026.04-rc1
make CROSS_COMPILE=aarch64-linux-gnu- rock-4d-rk3576_defconfig
export BL31=$(pwd)/../../tfa/build/rk3576/debug/bl31/bl31.elf
export TEE=$(pwd)/../../optee_os/out/arm-plat-rockchip/core/tee.bin
export ROCKCHIP_TPL=<path-to-rkbin>/bin/rk35/rk3576_ddr_lp4_2112MHz_lp5_2736MHz_v1.09.bin
make CROSS_COMPILE=aarch64-linux-gnu- -j$(nproc)
```

---

## Flash (SPI via MASKROM)

```bash
rkdeveloptool ld
rkdeveloptool db <rk3576_spl_loader>.bin
rkdeveloptool ef
rkdeveloptool wl 0 build/u-boot/u-boot-rockchip-spi.bin
rkdeveloptool rd
```

---

## Linux DT overlay (Armbian)

```bash
# Compile
dtc -@ -I dts -O dtb -o rk3576-optee.dtbo out/rk3576-optee.dts

# Install
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
