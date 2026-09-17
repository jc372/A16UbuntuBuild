# scripts/ — what each script is, and when it is used

All scripts are plain shell (or Python) with no hidden state; each logs to `~/a16-payload/` unless it
says otherwise, and each is safe to run twice.

## Installer media (WSL, Phase 1)

| Script | Used for |
|---|---|
| `run-wsl-build.sh` | the launcher: fetches linux-next, builds the kernel with `config/`, packages `zenbook-a16-<version>.tar.zst`. Modes: `kernel`, `gui`, `both`, `ubuntu-baseline` |
| `build.sh` | the build itself (called by the launcher) |
| `apply-series.sh` | applies the A16 mailing-list series to the tree, once, with a forward/reverse check; driven by `config/series.env` |
| `audit-config.sh` | checks the built configuration for the options this machine needs |
| `package.sh` | assembles the bundle (kernel, initrd, dtbs, modules, `metadata/`) |
| `make-ubuntu-desktop-usb-iso.sh` | remasters the Ubuntu daily around the bundle: kernel/initrd/DTB into `/casper`, Fedora Rawhide `qcom-firmware` + `atheros-firmware`, a GRUB menu with diagnostics, and the `65a16-live-root` casper-bottom hook |
| `make-ubuntu-daily-baseline.sh` | reproduces the untouched daily byte-for-byte, for comparison when a remaster misbehaves |
| `a16-harvest.sh` | the live-session collector: writes ACPI tables, `/proc/iomem`, device lists and `dmesg` to a FAT volume and prints a summary to the panel. Installed into the live root by the casper-bottom hook |

## Boot repair (live session, Phase 3–4)

| Script | Used for |
|---|---|
| `a16-finish-boot.sh` | **stage 1**, copied to the ESP as `\A16FIX.SH`: `grub-install --no-nvram --removable` into the installed system, `update-grub`, log to the ESP as `A16BOOT.LOG` |
| `a16-stage-esp-boot.sh` | **stage 2**, copied to the ESP as `\A16STAGE2.SH`: diagnostics plus a boot payload that needs only the ESP (`\a16boot\`), with a four-entry menu written to all four config locations GRUB may read |

## On the installed machine (Phase 5)

| Script | Used for |
|---|---|
| `a16-install-next-kernel.sh` | installs the built kernel + modules + device tree into the installed system |
| `a16-stage-dt-boot.sh` | writes the device-tree boot entries: `acpi=off`, the cleanup flags, the display blacklist for the fallback option (see `docs/boot-options.md`) |
| `a16-install-firmware.sh` | the four Qualcomm DSP blobs into `/lib/firmware/qcom/glymur/ASUSTeK/UX3607OA/` |
| `a16-bt-setup.sh` | Bluetooth firmware into `/lib/firmware/qca/`; `status` reports which device tree is live |
| `a16-bt-arm.sh` | arms the patched device tree over both stock DTB paths (stock kept as `*.a16stock`) |
| `a16-bt-dtb.sh` | rebuilds the Bluetooth device tree from the stock one (the polarity change) |
| `make-a16-qcc2072-board-2.sh` | rebuilds and installs the Wi-Fi `board-2.bin` for this machine's key |
| `a16-triage.sh` | quick probe of the Wi-Fi part: PCI ids, bound driver, firmware directories |
| `a16-drm-debug-entry.sh` | adds/removes kernel parameters on the boot entry, editing **all four** ESP menu configs |
| `a16-grub-dedupe.sh` | removes a duplicated menu entry that an old script appended repeatedly |

## Display, GPU, and module builds

| Script | Used for |
|---|---|
| `a16-build-gpucc-native.sh` | builds the missing GPU clock controller module (`gpucc-glymur`, `gxclkctl-kaanapali`) natively, harvesting the kernel's symbol CRCs into `Module.symvers`; refuses to build without `pahole` and verifies the module ABI afterwards |
| `a16-install-gpucc-module.sh` | installs built modules into `updates/`, refusing anything whose vermagic or `module_layout` CRC does not match the running kernel |
| `a16-fix-build-config.sh` | makes the build tree's config match the running kernel (pahole, `olddefconfig`, `syncconfig`) and verifies the ABI with `gdb` + `btf` |
| `a16-bootstrap.sh` | the whole machine state in one script: `--check`, `--tools`, `--tree`, `--patches`, `--config`, `--build`, `--install`, `--options`, `--all` |
| `a16-edp-debug.sh` | turns on `drm.debug` and re-runs eDP link training, collecting the result |
| `a16-gpu-param-probe.py` | issues the GPU parameter ioctl that used to oops the kernel; used before starting a desktop |
| `a16-gpu-fix.sh` | install a rebuilt `msm`, verify it, then start the desktop (the operator's one-command loop) |

## Evidence and monitoring

| Script | Used for |
|---|---|
| `a16-boot-report.sh`, `a16-boot-snapshot.sh`, `a16-enable-boot-snapshot.sh` | per-boot evidence: a snapshot of display/DRM/GPU state into a file the operator can read, taken automatically a short way into each boot |
| `a16-power-watch.sh` | samples the battery gauge over time |
| `render-docs.py` | inlines the patches into the `docs/` pages (`<!-- include: -->` markers), so a page always carries its own patch |

## Machine-specific paths

These scripts were written for the machine they brought up, so a few defaults are literal paths from
that machine:

- repository and payload: `~/A16UbuntuBuild`, `~/a16-payload/`
- build tree: `~/build/linux-next-<commit>`
- partition numbers: the repair scripts mount `/dev/nvme0n1p12` as the installed root, which is what
  it was at the time; on this machine today the root is `/dev/nvme0n1p17` and `p12` is the ESP.
  `findmnt -no SOURCE /` is the check

Each script that matters takes an environment override (`A16_TREE`, `A16_BUILD_DIR`, `A16_REPO`,
`A16_LOG`, `A16_PARAMS`, …) — check the top of the script before running it on different hardware,
and edit the one or two literals if needed. `scripts/a16-bootstrap.sh` derives the repository path
from its own location and is the recommended entry point.
