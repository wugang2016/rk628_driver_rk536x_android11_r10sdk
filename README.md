# README Languages / 文档语言

- [English](README.md) (current)
- [中文](README_zh.md)

---

# rk628_driver_rk536x_android11_r10sdk

## Overview

This repository contains the RK628 display chip driver, ported from **Linux kernel 6.1** to **Linux kernel 4.19**, targeting the **RK3566/RK356X Android 11 R10 SDK**.

The RK628 series chip (including RK628H) is a versatile display bridge chip developed by Rockchip, supporting HDMI-IN, BT1120, MIPI-DSI, and other input/output interfaces. This driver enables HDMI-IN functionality on the RK3566 platform.

## Background

The Rockchip Android 11 R10 SDK was released in 2021 with Linux kernel 4.19. Rockchip officially maintained RK628 driver version **v27-240730** as the last version compatible with the R10 SDK. Rockchip FAE clearly stated that Rockchip no longer maintains the RK628 driver for kernel 4.19, and all development has been moved to **kernel 5.10 / kernel 6.1** — requiring us to port the driver to the lower version ourselves.

Kernel 4.19 has reached end-of-life and is no longer maintained by Rockchip for this chip. Therefore, a backport was necessary.

### Original Problem: HDMI-IN Signal Recognition Instability

While using the RK628H for HDMI-IN on a RK3566 Android 11 R10 SDK device paired with a large-screen all-in-one machine, the HDMI-IN signal recognition was highly unstable:

- The HDMI-IN could sometimes capture video, but often failed to detect any signal after boot.
- After unplugging and re-plugging the HDMI cable, the signal would often fail to recover permanently.
- The issue was **100% reproducible** — unplugging and re-plugging the HDMI cable always triggered the problem.

**Error log pattern observed:**
```
[ 627.774866] android_work: sent uevent USB_STATE=CONNECTED
[ 629.588864] m00_b_rk628-csi 5-0050: rk628_hdmirx_phy_setup hdmi rxphy lock failed, retry:4
[ 631.625058] m00_b_rk628-csi 5-0050: rk628_hdmirx_phy_setup hdmi rxphy lock failed, retry:0
...
[ 633.544840] m00_b_rk628-csi 5-0050: no signal but 5v_det, recfg hdmirx!
```
The chip reported "no signal but 5v_det" and entered a retry loop, never recovering.

### Root Cause

Rockchip FAE confirmed that:
- The RK628 driver version v27-240730 was already the **last version** based on kernel 4.19.
- All subsequent development has been done on **kernel 5.10 / kernel 6.1**.
- The HDMI-IN signal stability issue required **updating the EQ (Equalizer) update strategy** in the PHY layer, which had been improved in newer kernel versions.
- Rockchip recommended upgrading the entire RK628 driver codebase from kernel 6.1.

Since upgrading the product SDK was not feasible (it is in mass production), the decision was made to **port the kernel 6.1 RK628 driver back to kernel 4.19**.

### Resolution

Rockchip provided the kernel 6.1 driver source code (`linux-src260527.tar.gz`, `rk628-src260527.tar.gz`, `cif-src260604.tar.gz`), and the driver was backported to kernel 4.19. Key adaptations included:

- Replacing kernel 6.1-specific CIF/CSI2 subsystems with the kernel 4.19 equivalents (`rkcif-externel.h`, `rockchip-csi2-dphy`, etc.).
- Adapting V4L2 subdev API differences between kernel 6.1 and kernel 4.19.
- Adapting clock management, GPIO, and media subsystem API changes.

The ported driver compiles successfully. Basic function tests are normal. HDMI-IN signal stability issue is pending further verification.

## Driver Architecture

```
rk628/
├── Kconfig               # Kernel config options
├── Makefile              # Build rules
├── rk628.c               # Core driver (probe, i2c ops)
├── rk628.h               # Core driver header
├── rk628_csi.h           # CSI interface definitions
├── rk628_csi_v4l2.c      # CSI/V4L2 subdev implementation
├── rk628_hdmirx.c        # HDMI-RX protocol layer
├── rk628_hdmirx.h        # HDMI-RX header
├── rk628_combrxphy.c     # Combined RX PHY
├── rk628_combrxphy.h
├── rk628_combtxphy.c     # Combined TX PHY
├── rk628_combtxphy.h
├── rk628_cru.c           # Clock/Reset Unit
├── rk628_cru.h
├── rk628_dsi.c           # DSI host driver
├── rk628_dsi.h
├── rk628_bt1120_v4l2.c   # BT1120 V4L2 driver
├── rk628_mipi_dphy.c     # MIPI D-PHY driver
├── rk628_mipi_dphy.h
├── rk628_post_process.c   # Post-processing (scaling, color space)
└── rk628_post_process.h
```

## Supported Chips and Features

| Chip       | Role         | Interfaces                  | Status |
|------------|--------------|-----------------------------|--------|
| RK628F/H   | HDMI/BT1120  | HDMI-IN, BT1120-IN, DSI-OUT | OK     |
| RK628C     | CSI Capture  | CSI-OUT                     | OK     |

## Porting Core

The porting is controlled by two macros defined in `rk628.h`:

```c
#define BJ_WUGANG_DEV  1

#if BJ_WUGANG_DEV
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(4, 19, 0))
#define KERNEL_VERSION_4_19
#endif
#if (LINUX_VERSION_CODE >= KERNEL_VERSION(5, 10, 0))
#define KERNEL_VERSION_5_10
#endif
#endif
```

- **`BJ_WUGANG_DEV`**: Porting adaptation switch — enables kernel-version-conditional compilation
- **`KERNEL_VERSION_4_19`**: Defined when kernel >= 4.19.0; the driver uses `#ifdef KERNEL_VERSION_4_19` branches to select between 4.19 and 6.1 API implementations

Key adaptation points:
- **`rk628_csi_v4l2.c`**: Uses `IS_REACHABLE(CONFIG_VIDEO_ROCKCHIP_CIF) && (!BJ_WUGANG_DEV)` to distinguish kernel 6.1's rkcif subsystem from kernel 4.19 compatibility paths; `#ifdef KERNEL_VERSION_4_19` branches cover PHY initialization, clock configuration, soft-reset, and more
- **`rk628_hdmirx.c`**: Uses `BJ_WUGANG_DEV` and `KERNEL_VERSION_4_19` to control HDMI-RX PHY configuration and EQ strategy

## DTS Configuration Example

```dts
rk628_csi_v4l2: rk628_csi_v4l2@50 {
    compatible = "rockchip,rk628-csi-v4l2";
    reg = <0x50>;
    i2s-enable-default;
    status = "okay";
    interrupt-parent = <&gpio0>;
    interrupts = <15 IRQ_TYPE_LEVEL_HIGH>;
    enable-gpios = <&gpio3 RK_PB6 GPIO_ACTIVE_HIGH>;
    reset-gpios = <&gpio0 RK_PB6 GPIO_ACTIVE_LOW>;
    plugin-det-gpios = <&gpio0 23 GPIO_ACTIVE_HIGH>;
    continues-clk = <1>;
    dynamic-eq;
    rockchip,camera-module-index = <0>;
    rockchip,camera-module-facing = "back";
    rockchip,camera-module-name = "RK628-CSI";
    rockchip,camera-module-lens-name = "NC";
    port {
        hdmiin_out0: endpoint {
            remote-endpoint = <&mipi_in>;
            data-lanes = <1 2 3 4>;
        };
    };
};
```

## Build Instructions

Place the `rk628/` directory under `kernel/drivers/media/i2c/` in the RK3566 Android 11 R10 SDK source tree, then enable the driver via `make menuconfig`:

```
Device Drivers  --->
    Multimedia support  --->
        Media drivers  --->
            I2C Encoders and sensors  --->
                <M> Rockchip rk628 display bridge
```

Or enable via Kconfig:
```
CONFIG_RK628=m
```

Then build the kernel as usual.

## Debug Interface

EQ register status can be read via debugfs:
```bash
cat /sys/kernel/debug/5-0050/registers/hdmirxphy | grep -E "0x34|0x54|0x74"
```

## License

GPL v2

## Acknowledgements

This project was developed by porting the **kernel 6.1** RK628 driver source code provided by Rockchip FAE team (`linux-src260527.tar.gz`, `rk628-src260527.tar.gz`, `cif-src260604.tar.gz`) back to kernel 4.19.
