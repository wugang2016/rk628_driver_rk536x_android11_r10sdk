# README Languages / 文档语言

- [English](README.md)
- [中文](README_zh.md) (current)

---

# rk628_driver_rk536x_android11_r10sdk

## 项目简介

本仓库包含 RK628 显示桥接芯片的驱动代码，已从 **Linux kernel 6.1** 回移植至 **Linux kernel 4.19**，适用于 **RK3566/RK356X Android 11 R10 SDK**。

RK628 系列芯片（含 RK628H）是瑞芯微（Rockchip）推出的多功能显示桥接芯片，支持 HDMI-IN、BT1120、MIPI-DSI 等多种输入输出接口。本驱动在 RK3566 平台上实现了 HDMI-IN（HDMI 输入）功能。

## 移植背景

Rockchip Android 11 R10 SDK 发布于 2021 年，内核版本为 Linux kernel 4.19。Rockchip 官方维护的 RK628 驱动版本 **v27-240730** 是能支持 R10 SDK 的最后一版。Rockchip FAE 明确告知，当前 Rockchip 已不再维护 kernel 4.19 版本上的 RK628 驱动，仅在 **kernel 5.10 / kernel 6.1** 上继续开发，要求我司自行完成低版本的移植工作。

kernel 4.19 已进入维护末期，Rockchip 也不再为该芯片的 4.19 内核版本提供后续支持。因此需要自行完成从 kernel 6.1 到 kernel 4.19 的驱动回移植。

### 原始问题：HDMI-IN 信号识别不稳定

在我司 RK3566 Android 11 R10 SDK 设备上使用 RK628H 做 HDMI-IN 功能，与鸿合某大屏一体机配套使用时，出现 HDMI-IN 信号识别不稳定的问题：

- 有时 HDMI-IN 能采集到画面，但多数时候启动后死活无法识别信号；
- 有时插拔一次 HDMI 线后就再也无法获取图像，此后一直无法识别 HDMI-IN 信号；
- 该问题**必现**，插拔 HDMI 线后必定触发。

**日志中观察到的错误模式：**
```
[ 627.774866] android_work: sent uevent USB_STATE=CONNECTED
[ 629.588864] m00_b_rk628-csi 5-0050: rk628_hdmirx_phy_setup hdmi rxphy lock failed, retry:4
[ 631.625058] m00_b_rk628-csi 5-0050: rk628_hdmirx_phy_setup hdmi rxphy lock failed, retry:0
...
[ 633.544840] m00_b_rk628-csi 5-0050: no signal but 5v_det, recfg hdmirx!
```
芯片报"有 5V 检测但无信号"（no signal but 5v_det），持续重试但始终无法恢复。

### 问题根因

与 Rockchip FAE 团队沟通后确认：

- RK628 驱动 v27-240730 已经是**基于 kernel 4.19 的最后一版**；
- 之后 Rockchip 所有 RK628 驱动的开发均基于 **kernel 5.10 / kernel 6.1**；
- 该 HDMI-IN 信号稳定性问题需要**更新 PHY 层 EQ（Equalizer）更新策略**，这部分在新版内核中有改进；
- Rockchip 建议升级整个 RK628 代码包，而非仅针对 EQ 部分做修改。

由于产品正在大规模商用阶段，升级 SDK 改动量过大、风险高，我司决定将 **kernel 6.1 的 RK628 驱动整体回移植到 kernel 4.19**。

### 解决过程

Rockchip FAE 提供了 kernel 6.1 的驱动源码包（`linux-src260527.tar.gz`、`rk628-src260527.tar.gz`、`cif-src260604.tar.gz`），我司在此基础上完成了驱动回移植。移植过程中主要解决了以下适配问题：

- **V4L2 subdev API 差异**：kernel 版本间 subdev 操作 API 有差异，逐个适配；
- **时钟管理、GPIO、媒体子系统 API 变更**：确保各接口在 kernel 4.19 下的兼容性。

> **CIF 适配说明**：R10 SDK 的 CIF 版本较老，API 与 kernel 6.1 不兼容，`rkcif-externel.h` 接口在本移植中**不会被调用** — 代码中保留了 CIF 相关适配分支，但在 R10 SDK 目标平台上实际并未启用。

移植后驱动编译通过，基本功能测试正常，HDMI-IN 信号识别不稳定问题待验证。

## 驱动代码结构

```
rk628/
├── Kconfig               # 内核配置选项
├── Makefile              # 编译规则
├── rk628.c               # 核心驱动（probe、I2C 操作）
├── rk628.h               # 核心驱动头文件
├── rk628_csi.h           # CSI 接口定义
├── rk628_csi_v4l2.c      # CSI/V4L2 subdev 实现（移植主要改动点）
├── rk628_hdmirx.c        # HDMI-RX 协议层
├── rk628_hdmirx.h        # HDMI-RX 头文件
├── rk628_combrxphy.c     # 组合 RX PHY
├── rk628_combrxphy.h
├── rk628_combtxphy.c     # 组合 TX PHY
├── rk628_combtxphy.h
├── rk628_cru.c           # 时钟/复位管理单元
├── rk628_cru.h
├── rk628_dsi.c           # DSI 主机驱动
├── rk628_dsi.h
├── rk628_bt1120_v4l2.c   # BT1120 V4L2 驱动
├── rk628_mipi_dphy.c     # MIPI D-PHY 驱动
├── rk628_mipi_dphy.h
├── rk628_post_process.c  # 后处理（缩放、色彩空间转换）
└── rk628_post_process.h
```

## 支持的芯片及功能

| 芯片       | 角色          | 接口                        | 状态 |
|------------|---------------|-----------------------------|------|
| RK628F/H   | HDMI/BT1120   | HDMI-IN、BT1120-IN、DSI-OUT | 正常 |
| RK628C     | CSI 采集      | CSI-OUT                     | 正常 |

## 移植核心说明

本移植的核心通过两个宏控制，集中在 `rk628.h` 中定义：

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

- **`BJ_WUGANG_DEV`**：移植适配开关，开启后根据当前内核版本条件编译对应代码
- **`KERNEL_VERSION_4_19`**：内核 >= 4.19.0 时定义，驱动内通过 `#ifdef KERNEL_VERSION_4_19` 分支选择 4.19 或 6.1 的 API 实现

主要适配点：
- **`rk628_csi_v4l2.c`**：通过 `IS_REACHABLE(CONFIG_VIDEO_ROCKCHIP_CIF) && (!BJ_WUGANG_DEV)` 区分 kernel 6.1 的 rkcif 子系统和 kernel 4.19 的兼容性处理，`#ifdef KERNEL_VERSION_4_19` 分支覆盖 PHY 初始化、时钟配置、软复位等多个子模块
- **`rk628_hdmirx.c`**：通过 `BJ_WUGANG_DEV` 和 `KERNEL_VERSION_4_19` 控制 HDMI-RX PHY 配置及 EQ 策略

## DTS 配置示例

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

## 编译说明

将 `rk628/` 目录放入 RK3566 Android 11 R10 SDK 源码树的 `kernel/drivers/media/i2c/` 目录下，通过 `make menuconfig` 启用驱动：

```
Device Drivers  --->
    Multimedia support  --->
        Media drivers  --->
            I2C Encoders and sensors  --->
                <M> Rockchip rk628 display bridge
```

或在对应 Kconfig 中直接添加：
```
CONFIG_RK628=m
```

然后正常编译内核即可。

## 调试手段

通过 debugfs 查看 EQ 寄存器状态：
```bash
cat /sys/kernel/debug/5-0050/registers/hdmirxphy | grep -E "0x34|0x54|0x74"
```

正常情况下这三个寄存器的值应反映 DTS 中配置的 EQ 策略是否生效。

## 开源许可

GPL v2

## 致谢

本项目基于 Rockchip FAE 团队提供的 **kernel 6.1** RK628 驱动源码包（`linux-src260527.tar.gz`、`rk628-src260527.tar.gz`、`cif-src260604.tar.gz`）完成移植。
