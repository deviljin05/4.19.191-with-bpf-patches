# X6815C Kernel Builder (BPF)

Automated GitHub Actions pipeline that builds a flashable **Linux 4.19.191** kernel for the **Infinix Zero 5G 2023 (X6815C, MediaTek Dimensity 920 / MT6877)** with **eBPF support enabled and verified**, and packages it as an **AnyKernel3** recovery-flashable ZIP.

![Platform](https://img.shields.io/badge/platform-arm64-blue)
![Kernel](https://img.shields.io/badge/kernel-4.19.191-orange)
![Toolchain](https://img.shields.io/badge/clang-r383902%20(11.0.1)-green)
![License](https://img.shields.io/badge/license-GPL--2.0-lightgrey)

---

## Table of contents

- [Overview](#overview)
- [Features](#features)
- [Target device](#target-device)
- [Quick start](#quick-start)
- [Workflow inputs](#workflow-inputs)
- [Build outputs](#build-outputs)
- [BPF support](#bpf-support)
- [Patches](#patches)
- [Firmware files](#firmware-files)
- [Flashing](#flashing)
- [Repository layout](#repository-layout)
- [Troubleshooting](#troubleshooting)
- [Credits and license](#credits-and-license)
- [Disclaimer](#disclaimer)

---

## Overview

This repository contains the CI workflow (`.github/workflows/build.yml`) that clones the
[`android_kernel_infinix_X6815C`](https://github.com/rag12ghu-design/android_kernel_infinix_X6815C)
kernel source, configures it, compiles it with the same compiler family used for the stock kernel, and
publishes a ready-to-flash ZIP together with the final `.config`.

The workflow is designed to **fail early and explain why**: configuration invariants and BPF options are
checked before the long compile starts, and the first errors of a failed build are written to the job summary.

## Features

- **One-click builds** through *Actions → Run workflow*, no local toolchain needed.
- **Stock-compatible compiler**: AOSP Clang `r383902` (11.0.1) with ThinLTO and CFI, as used by the stock kernel.
  Proton-Clang is available as a fallback.
- **BPF enabled and verified**: required options are checked after `olddefconfig`; the build stops before compiling if any of them did not stick.
- **ABI safety checks**: the build refuses to continue if Shadow Call Stack is enabled, or if LTO, CFI, modules or modversions are off, because the prebuilt vendor modules depend on this layout.
- **Device-specific make variables** (MediaTek connectivity, WLAN, Bluetooth) are passed to every `make` call through one shared wrapper.
- **Known source issues fixed automatically** before the build: Kconfig and Makefile errors that produced warnings, and the missing built-in touchscreen firmware.
- **Flexible patching**: optional patch folder, patch URLs and AOSP cherry-picks, all applied before configuration.
- **ccache** with restore/save, so repeat builds are much faster, even after a failed run.
- **AnyKernel3 packaging** using the template shipped in the kernel repository (device names, banner and dtb already set up).
- **Optional GitHub Release** with the ZIP and `final.config` attached.

## Target device

| Item | Value |
| --- | --- |
| Device | Infinix Zero 5G 2023 (`X6815C`) |
| SoC | MediaTek Dimensity 920 (MT6877) |
| Architecture | arm64 |
| Kernel | Linux 4.19.191, not GKI |
| Defconfig | `X6815C_defconfig` |
| Compiler | AOSP Clang r383902 (11.0.1), ThinLTO + CFI |
| Boot image | Android boot header v2, kernel + stock dtb |

## Quick start

1. **Fork or create** a repository and add this workflow as `.github/workflows/build.yml`.
2. Open the **Actions** tab, select **Build X6815C Kernel (BPF)** and click **Run workflow**.
3. Leave the defaults for the first run. Defaults build the `main` branch with the AOSP toolchain and BPF enabled.
4. When the run finishes, download the artifact `X6815C-build-<run number>` from the run page.
5. Read the **job summary**: it lists the config checks and, on failure, the first compiler errors.

A full build takes a while on the standard GitHub runner. Later builds are faster thanks to ccache.

## Workflow inputs

| Input | Default | Description |
| --- | --- | --- |
| `kernel_repo` | `https://github.com/rag12ghu-design/android_kernel_infinix_X6815C` | Kernel source repository. |
| `kernel_branch` | `main` | Branch to build. |
| `defconfig` | *(empty)* | Defconfig name. Empty selects `X6815C_defconfig`. |
| `toolchain` | `aosp` | `aosp` (Clang r383902) or `proton` (fallback, may change the module ABI). |
| `enable_bpf` | `true` | Enable BPF options and fail early if they do not stick. |
| `include_dtb` | `false` | Replace the dtb in the ZIP with the one built from source. Off keeps the dtb shipped in the repository's AnyKernel3. |
| `include_modules` | `false` | Ship built `.ko` files in the ZIP (experimental). Off keeps the stock vendor modules. |
| `patch_urls` | *(empty)* | Raw `.patch` / `.diff` URLs, separated by spaces or new lines. |
| `cherry_picks` | *(empty)* | Commit SHAs from AOSP `kernel/common` (`android-4.19-stable`), space separated. |
| `release` | `false` | Publish a GitHub Release. |

## Build outputs

Each run uploads two artifacts:

- **`X6815C-build-<run>`**
  - `X6815C-<version>[-BPF]-run<N>.zip`: the flashable AnyKernel3 package
  - `Image.gz`: the compiled kernel image
  - `final.config`: the resolved kernel configuration
  - `Module.symvers`: exported symbols and CRCs, useful for checking module compatibility
- **`build-log-<run>`**: the full compiler log

## BPF support

The workflow enables and checks the following options. The first group must be set; the second group only produces a warning.

| Required | Optional |
| --- | --- |
| `BPF`, `BPF_SYSCALL`, `BPF_JIT` | `BPF_JIT_ALWAYS_ON` |
| `CGROUP_BPF` | `BPF_EVENTS` |
| `NET_CLS_BPF`, `NET_ACT_BPF`, `NET_CLS_ACT`, `NET_SCH_INGRESS` | |
| `NETFILTER_XT_MATCH_BPF` | |

The kernel source already contains the BPF core (`kernel/bpf/`), the traffic-control classifiers and the
`xt_bpf` netfilter match, so **no source patches are required**. The workflow only turns on the options that
the stock defconfig leaves off (for example `NET_ACT_BPF`). Results are written to the GitHub job summary.

## Patches

Patches are optional. Three mechanisms are available, applied in this order:

1. **`patches/bpf/*.patch`** in this repository. Applied alphabetically when `enable_bpf` is on. A patch that does not apply cleanly is skipped with a warning.
2. **`patch_urls`** input. Downloaded and applied in order. A failure stops the build.
3. **`cherry_picks`** input. Commits are fetched from `https://android.googlesource.com/kernel/common` and applied. A failure stops the build.

To create a patch from your own changes:

```bash
git clone https://github.com/rag12ghu-design/android_kernel_infinix_X6815C
cd android_kernel_infinix_X6815C
# edit files
git diff > 0001-my-change.patch
```

## Firmware files

The kernel defconfig lists `novatek_ts_fw.bin` as built-in firmware, but `*.bin` files are excluded by the
kernel repository's `.gitignore`. If a listed firmware file is missing, the workflow removes it from
`CONFIG_EXTRA_FIRMWARE` and prints a warning, so the build can continue. The touchscreen driver then requests
the file at runtime when it needs it.

To embed your own firmware, create a **`firmware/`** folder in this repository and add the files there. They are
copied into the kernel source before configuration.

## Flashing

> **Back up your current `boot` partition before flashing anything.**

1. Download the ZIP from the artifact or Release page.
2. Boot into a custom recovery that can flash ZIP files.
3. Flash the ZIP. AnyKernel3 patches the kernel in your existing boot image and keeps your ramdisk.
4. Reboot, and verify with `uname -a` and `zcat /proc/config.gz | grep BPF`.

If the device does not boot, restore the backed-up `boot` image with fastboot or your recovery.

## Repository layout

```text
.
├── .github/workflows/build.yml   # the build pipeline
├── patches/bpf/                  # optional BPF patches (*.patch)
├── firmware/                     # optional firmware files to embed
└── README.md
```

## Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `No rule to make target 'firmware/...bin'` | A built-in firmware file is missing. The workflow removes it automatically. To embed it, add it to `firmware/`. |
| `CONFIG_<name> did not stick after olddefconfig` | The option is missing in Kconfig or has an unmet dependency. The log shows which. A patch may be needed. |
| Config check fails on `SHADOW_CALL_STACK`, `LTO_CLANG` or `CFI_CLANG` | The defconfig or a fragment changed the stock ABI layout. Restore the original values. |
| `AOSP clang ... could not be downloaded` warning | The workflow fell back to Proton-Clang. The kernel may build, but the module ABI can differ from stock. |
| Patch does not apply | The patch targets a different tree. Rebuild it against the 4.19.191 source. |
| `LLVM ERROR: IO failure on output stream: Broken pipe` | Harmless noise from Makefile probes. It is filtered from the build log. |
| Out of disk space | The workflow already frees space on the runner. Disable `include_modules` and `include_dtb`. |

When reporting a problem, attach the `build-log-<run>` artifact and the first errors from the job summary.

## Credits and license

- Kernel source: [`rag12ghu-design/android_kernel_infinix_X6815C`](https://github.com/rag12ghu-design/android_kernel_infinix_X6815C), based on the Linux kernel and MediaTek / Transsion vendor sources.
- Packaging: [AnyKernel3](https://github.com/osm0sis/AnyKernel3) by osm0sis and contributors.
- Toolchain: AOSP prebuilt Clang; fallback [Proton-Clang](https://github.com/kdrag0n/proton-clang) by kdrag0n.

The Linux kernel is licensed under **GPL-2.0**. Please follow the licenses of all bundled components.

## Disclaimer

Flashing a custom kernel can cause boot loops, data loss or void your warranty. You do this at your own risk.
This project is provided as is, without any warranty, and is not affiliated with Infinix, Transsion or MediaTek.
