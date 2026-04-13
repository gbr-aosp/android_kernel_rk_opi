# Radxa ROCK 5B+ Kernel Port

Porting the dvab-sarma Android kernel (based on `common-android16-6.12-lts`) to support the
Radxa ROCK 5B+ (RK3588). The original kernel tree targets Orange Pi 5 series boards.

Source manifest: https://github.com/gbr-aosp/android_kernel_manifest

## What Was Changed

### 1. Device Tree Source (`rk3588-rock-5b-plus.dts`)

**File:** `common/arch/arm64/boot/dts/rockchip/rk3588-rock-5b-plus.dts`

A new DTS was created for the Rock 5B+, based on the existing upstream-style
`rk3588-rock-5b.dts` already present in this ACK tree. Key hardware differences
from the Rock 5B that are reflected in this DTS:

- **PCIe 3.0 lane split** — `data-lanes = <1 1 2 2>` on `pcie30phy`, splitting
  4 lanes into 2+2: two for NVMe via `pcie3x4` (set to `num-lanes = <2>`) and
  two for the M.2 B-key slot via `pcie3x2`.
- **WWAN / LTE support** — `rfkill-wwan` node and GPIO hogs on `gpio0` / `gpio2`
  for M.2 B-key W_DISABLE, RESET, and WoWWAN signals.
- **USB-C with full PD** — FUSB302 type-C controller on `i2c4` with USB-C
  connector supporting dual power-role (sink/source), DisplayPort alt-mode via
  `usbdp_phy0`, and SBU DC detect GPIOs.
- **NPU regulator** — `vdd_npu_s0` on `i2c1` (RK8602) for the dedicated NPU
  power rail, plus I2C EEPROM for board identification.
- **USB host power GPIO** — `vcc5v0_host` uses GPIO1_PA1 (vs GPIO4_PB0 on Rock 5B).
- **WWAN power regulator** — `vcc3v3_wwan_pwr` controlled via GPIO2_PB1.

The Radxa BSP DTS was **not** used directly because it depends on Rockchip SDK-specific
includes (`rk3588-rk806-single.dtsi`, `rk3588-linux.dtsi`) and vendor-proprietary
bindings that are not present in this ACK/GKI kernel tree. The BSP DTS was used as a
hardware reference to verify GPIO assignments and peripheral configuration.

### 2. Build Configuration (`build.config.rock5bplus`)

**File:** `common/build.config.rock5bplus`

```
KERNEL_DIR=common
BRANCH=
DEFCONFIG=android_orangepi5_defconfig
FILES=
IN_KERNEL_MODULES=
```

Uses the same `android_orangepi5_defconfig` as the Orange Pi boards, since all
boards share the RK3588 platform and this defconfig already enables
`CONFIG_ARCH_ROCKCHIP`, PCIe, USB, STMMAC Ethernet, Rockchip thermal, Mali GPU,
and all other RK3588 subsystems.

### 3. DTS Makefile Entry

**File:** `common/arch/arm64/boot/dts/rockchip/Makefile`

Added an uncommented (active) build entry:

```makefile
#radxa rock 5b plus rk3588
dtb-$(CONFIG_ARCH_ROCKCHIP) += rk3588-rock-5b-plus.dtb
```

This was placed before the Orange Pi entries. The existing `rk3588-rock-5b.dtb`
entry remains commented out (as it was upstream).

### 4. Bazel BUILD Targets

**File:** `common/BUILD.bazel`

Two new Kleaf `kernel_build` targets were added:

```starlark
# Defconfig target
kernel_build(
    name = "rock5bplus_defconfig",
    outs = [".config"],
    build_config = "build.config.rock5bplus",
    make_goals = ["android_orangepi5_defconfig"],
)

# Full kernel + DTB build
kernel_build(
    name = "rock5bplus",
    outs = [
        "arch/arm64/boot/Image",
        "arch/arm64/boot/dts/rockchip/rk3588-rock-5b-plus.dtb",
    ],
    build_config = "build.config.rock5bplus",
    make_goals = ["Image", "dtbs"],
    makefile = "//common:Makefile",
    kcflags = ["-D__ANDROID_COMMON_KERNEL__"],
)
```

These follow the same pattern as the existing `opi5_pro`, `opi5`, and `opi3b` targets.

## How to Build

### Prerequisites

1. Establish an [Android build environment](https://source.android.com/setup/initializing)
   and install [repo](https://source.android.com/docs/setup/develop#installing-repo).

2. Initialize and sync the kernel source:

```bash
repo init -u https://android.googlesource.com/kernel/manifest -b common-android16-6.12-lts
curl -o .repo/local_manifests/manifest_rk_opi.xml -L \
  https://raw.githubusercontent.com/gbr-aosp/android_kernel_manifest/android-16.0/manifest_rk_opi.xml \
  --create-dirs
repo sync
```

### Compile

```bash
tools/bazel build --config=fast --config=stamp //common:rock5bplus
```

### Output Artifacts

After a successful build, the artifacts are at:

| Artifact | Path |
|----------|------|
| Kernel Image | `bazel-bin/common/rock5bplus/Image` |
| Device Tree Blob | `bazel-bin/common/rock5bplus/rk3588-rock-5b-plus.dtb` |

### Clean Rebuild

If you need a clean rebuild:

```bash
cd common
make mrproper
cd ..
rm -rf out/ bazel-out/ bazel-bin/ bazel-* .bazelrc
tools/bazel clean --expunge
```

Then run the compile step again.

## File Summary

| File | Action | Description |
|------|--------|-------------|
| `common/arch/arm64/boot/dts/rockchip/rk3588-rock-5b-plus.dts` | Created | Rock 5B+ device tree |
| `common/build.config.rock5bplus` | Created | Kernel build configuration |
| `common/arch/arm64/boot/dts/rockchip/Makefile` | Modified | Added DTB build entry |
| `common/BUILD.bazel` | Modified | Added Bazel build targets |
