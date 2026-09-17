# A16UbuntuBuild — installing and bringing up Ubuntu on the ASUS Zenbook A16

Step-by-step instructions for installing Ubuntu 26.10 on an **ASUS Zenbook A16 UX3607OA**
(Qualcomm Snapdragon X2 Elite "Glymur", 48 GB) and bringing it up: internal display, Wi-Fi,
Bluetooth, keyboard/touchpad/touchscreen, battery. Windows 11 ARM64 stays installed on the same machine
throughout; it is both the fallback and the source of some firmware.

Only the procedure that works is included; the earlier experimental attempts are not.

## End state

| | |
|---|---|
| OS | Ubuntu 26.10, installed on the internal NVMe, booting on a device tree with `acpi=off` |
| Kernel | linux-next, `7.3.0-rc3-next-20260914` at the time of writing |
| Working | internal panel with backlight and 120 Hz, Wi-Fi, Bluetooth, keyboard, touchpad, touchscreen, stylus, battery/charge control |
| Not working | internal speakers (needs a machine ACPI topology), suspend, external displays (needs in-review PHY work) |

## What you need

**Install-time hardware — the internal keyboard and touchpad cannot be relied on during the
installation, and the USB-C ports do not work yet.**

| | |
|---|---|
| USB-A dock or hub | plugged into the machine's **USB-A** port, carrying the three devices below |
| Wired keyboard | for the installer and the live session |
| Wired mouse | same |
| Ethernet | through the dock — recommended, and the easiest way to get the clock right (see below) |
| Install media | a **USB stick**, or the machine's **SD card** |

**Do not plan on the USB-C ports for the initial install.** Their support (USB3, the combo PHY and
alternate mode) is not in place until the machine is patched in Phase 5; see
`docs/display-outputs.md` for the state of that work. The USB-A port is what carries the dock, so
the keyboard, mouse and Ethernet all come from that one connection.

**Why the wired input is not optional:** this machine's internal keyboard, touchpad and touchscreen
only work when Linux is booted on a device tree with `acpi=off` (see `docs/input.md`). A stock
installer media boots ACPI, where nothing internal matches a driver and there is no input at all.

**Why Ethernet matters:** the machine has no RTC that Linux can read, so a live session starts with
the wrong date, and Ubuntu's package indices carry a `Valid-Until` — a wrong clock breaks the
installer. Ethernet gives you NTP, which fixes it in one step (Phase 2 also gives the manual
alternative).

## Secure Boot

**Do this at every switch between the two systems — it is a standing step, not a one-off:**

| Booting | Secure Boot |
|---|---|
| the installer media, or the installed Linux | **disable** |
| Windows | **re-enable** |

Why: the Linux side runs a kernel and modules built here, which are **unsigned**, and Secure Boot
would refuse them; the Windows side expects the firmware's Secure Boot setting as it shipped.

In the firmware, at power-on:

| Key | What it opens |
|---|---|
| **F2** | the BIOS setup (Security → Secure Boot → Disabled / Enabled) |
| **Esc** | the boot options |

Leave Secure Boot disabled whenever you are working in Linux, and turn it back on before you hand the
machine over to Windows or finish.

## Disk layout of this machine

    nvme0n1p12   450M  vfat   EFI System Partition, label SYSTEM   mounted /boot/efi   UUID 7E07-8CF1
    nvme0n1p13    16M         Microsoft reserved
    nvme0n1p14 853.1G  BitLocker  Windows data ("ZEN OS 6/20/2026")
    nvme0n1p15   1.6G  ntfs   Windows recovery environment
    nvme0n1p16   260M  vfat   Windows recovery environment (MYASUS)
    nvme0n1p17  95.7G  ext4   Ubuntu root          UUID f8e005e9-414c-4c8e-ad68-d1e9fdc208bc
    nvme0n1p18     2G  swap   Ubuntu swap

**Partition numbers move.** The repair scripts in `scripts/` and `steps/` were run when the installed
root was a different partition number, and they say so internally. Always confirm before mounting
anything — mounting the ESP where the root was expected is the kind of mistake that is easy to make
and unpleasant to notice:

    findmnt -no SOURCE,UUID,FSTYPE /            # the installed root
    findmnt -no SOURCE,UUID,FSTYPE /boot/efi    # the ESP
    lsblk -o NAME,SIZE,FSTYPE,PARTTYPENAME,LABEL,MOUNTPOINT

## Firmware boot entries of this machine

    BootCurrent: 0001
    BootOrder:   0002,0001,0000,0003
    Boot0000* Ubuntu Linux   \EFI\ubuntu_snapdragon\grubaa64.efi
    Boot0001* Ubuntu         \EFI\ubuntu_snapdragon\shimaa64.efi
    Boot0002* Windows Boot Manager
    Boot0003* ubuntu         \EFI\ubuntu\shimaa64.efi

EFI variables **do** work once the installed system is running, even though they did not in the live
session during installation — that is why the boot entries above exist, and why the firmware menu
offers Windows and Linux side by side.

## Before you start

Three things about this machine decide the whole procedure:

1. **The installer cannot finish on its own.** Ubuntu's installer aborts at the *last* step,
   `install-grub`, because this machine gives Linux no EFI variables
   (`efibootmgr: EFI variables are not supported on this system`) — so nothing is written to the EFI
   partition and no firmware boot entry is created. The boot has to be completed by hand afterwards.
   This is expected, not a mistake, and Phase 3/4 exist for it.
2. **ACPI mode cannot give you the internal keyboard or touchpad** (`ACPI\QCOM0F10` and
   `ACPI\QCOM0F0C` match no driver), so the system is booted on a **device tree** with `acpi=off`.
   The device tree path is what makes internal input work at all.
3. **The clock.** This machine has no RTC as far as Linux is concerned, so the live session and a
   fresh install start with a wrong date, which breaks package downloads and the installer. Set the
   clock (or connect to a network) before starting the installer.

## Order of work

Phases 0 and 1 need no Linux on the machine, and Phase 0 needs no Linux at all. Do all of Phase 0
before you flash anything: it is where the firmware comes from, and it saves round trips later.

    Phase 0   Windows, in advance          downloads + firmware extraction + WSL setup      steps/00-prepare-in-advance.md
    Phase 1   WSL                          build the kernel + remaster the installer media  steps/01-build-installer-media.md
    Phase 2   live session                 install Ubuntu (expect the install-grub failure) steps/02-install-ubuntu.md
    Phase 3   Windows                      put the boot-repair scripts on the EFI partition steps/03-stage-repair-scripts.md
    Phase 4   live session                 finish the boot: stage 1, stage 2                 steps/04-finish-the-boot.md
    Phase 5   installed system             first boot, then bring the machine up            steps/05-bring-up.md
    Phase 6   after any kernel change      rebuild the modules with the config/ABI check    steps/06-module-builds.md

### Do these first

- Extract **all** Windows-side firmware before installing: the four Qualcomm DSP blobs, the Wi-Fi
  board-data source package, and the Bluetooth firmware files (Phase 0). Having them on a stick
  means the first boot has Wi-Fi and Bluetooth immediately.
- Put the two boot-repair scripts **on the EFI partition before running the installer**, from the
  live session or from Windows. The install failure is predictable, so the repair should not need a
  trip to Windows afterwards.
- Build the kernel and the installer media (Phase 1) while the machine is still usable in Windows —
  the build takes longer than the installation does.
- Set the clock as the first action of every live session.

## Phase 0 — Windows: downloads and firmware

Do this on the machine's own Windows 11 installation (or any Windows machine with access to the
DriverStore of this laptop).

1. **Downloads**

   - The **Ubuntu ARM64 desktop daily** image for the series you want. The one used here is the
     26.10 ("stonking") desktop daily, which boots on this machine and has working internal input.
     Verify its checksum (`Get-FileHash` in PowerShell, or `sha256sum`).
   - **Rufus 4.14** — used in **DD image mode**, with **Secure Boot off** (Rufus logs to
     `%USERPROFILE%\Downloads\Rufus\rufus.log`).
   - Optionally the **ASUS Qualcomm BSP** (`V1.312.4500.0`, sha256
     `E8F2389AC5D4DFD30A1A8481EC3C1E282F24BAEA2898414351D0B6EB1FA8A0B3`). It is *not* needed: it
     contains no SoCCP firmware (`soccp.mbn`/`soccp_dtb.mbn` are copied from the device's SPI-NOR at
     the factory). Do not synthesise them by renaming `RSCP.bin`.

2. **Firmware to extract from this machine's own Windows install**

   | What | Where it comes from | Goes to |
   |---|---|---|
   | `qcadsp8480.mbn`, `adsp_dtbs.elf`, `qccdsp8480.mbn`, `cdsp_dtbs.elf` | the signed ADSP and CDSP driver directories in the Windows DriverStore | `firmware/qcom/glymur/ASUSTeK/UX3607OA/` |
   | Wi-Fi board data for the QCC2072 (`17cb:1112`, SUBSYS `E14F105B`) | the vendor Wi-Fi package (its `bdwlan`/board files) | rebuilt with QCA's `bdencoder` into `board-2.bin` (sha256 `314e2d57…`) by `scripts/make-a16-qcc2072-board-2.sh` |
   | Bluetooth firmware (`hmtbtfw20.tlv`, `hmtnv20.b*`) | the vendor Bluetooth package | `/lib/firmware/qca/` (`scripts/a16-bt-setup.sh install`) |

   Copy the DSP blobs and the Wi-Fi/Bluetooth packages to a folder you will carry into Linux (a
   stick, or a copy on the Windows partition that Linux can mount read-only).

3. **Facts worth capturing while you are in Windows** — the device-to-driver inventory from the
   DriverStore, and the vendor package for each Qualcomm device. This is what identified the Wi-Fi
   part and its board data, and it is the only source for those blobs.

4. **WSL** (Windows Subsystem for Linux) with an Ubuntu distribution:

       wsl --install -d Ubuntu

   Then inside WSL, the build needs:

       sudo apt-get update
       sudo apt-get install -y xorriso mtools cpio rpm zstd xz-utils curl \
            coreutils findutils gzip grep sed gawk tar \
            bc bison flex openssl libelf-dev libdw-dev gcc make dtc ccache \
            python3 perl git pipx
       pipx install b4                # only needed when applying a mailing-list series by hand

   `scripts/run-wsl-build.sh` installs these itself for the Ubuntu/Debian, Fedora and openSUSE
   cases; see its `BUILD_PACKAGES` block.

## Phase 1 — WSL: build the kernel, then remaster the installer image

The stock daily's kernel does not drive this machine, so the installer media is rebuilt around a
linux-next kernel built with this machine's config. The kernel build inputs are in `config/`.

1. **Get the tree and build the payload.** The A16 support is in linux-next (the binding, the board
   DTS and the firmware allowlist landed upstream by 2026-08-14), so a plain linux-next checkout is
   enough:

       scripts/run-wsl-build.sh kernel            # fetches linux-next, applies nothing, builds, packages
       scripts/run-wsl-build.sh kernel --ref <tag-or-sha>

   The result is a bundle, `zenbook-a16-<version>.tar.zst`, containing the kernel, the initrd, the
   device trees and the modules, plus `metadata/` (config and the exact commit).

2. **Remaster the installer image** around that bundle:

       scripts/make-ubuntu-desktop-usb-iso.sh \
           <path>/zenbook-a16-<version>.tar.zst \
           https://cdimage.ubuntu.com/.../<series>-desktop-arm64.iso

   The remaster writes the bundle's `vmlinuz`, `initrd` and the A16 device tree into `/casper`, adds
   the current Fedora Rawhide `qcom-firmware` and `atheros-firmware` packages (unpacked with
   `rpm2cpio`), and installs a `casper-bottom` hook (`65a16-live-root`) that runs in the live
   session. It also writes a GRUB menu with diagnostic entries:

       acpi=off console=tty0 earlycon keep_bootcon loglevel=8 initcall_debug ... module_blacklist=msm

   The `module_blacklist=msm` on the diagnostic entries matters: with `msm` loaded on a machine whose
   display path was, at that point, untested, the console is what you want.

3. **Flash** with Rufus in **DD image mode**, Secure Boot off.

## Phase 2 — install Ubuntu (expect the boot failure)

1. Boot the stick. Choose the normal entry; the internal keyboard and touchpad work here.
2. **Set the clock first** (no RTC, and the installer's package indices carry a `Valid-Until`):

       sudo timedatectl set-ntp false
       sudo date -s '2026-09-16 10:30:00'      # today's real date -- not a date in the past
       date

   Connecting a network and letting NTP sync achieves the same thing.

3. Run the installer:

       ubuntu-desktop-bootstrap

4. **It will abort at the end**, in `install-grub`, with:

       efibootmgr: EFI variables are not supported on this system.

   That is this machine: Linux is given no EFI variables, `curthooks` fails, the EFI partition is
   left untouched and no firmware boot entry exists. The installed root filesystem, however, is
   complete. Do not reinstall — continue to Phase 3.

## Phase 3 — put the boot-repair scripts on the EFI partition

The two scripts that finish the boot are in `scripts/`:

| Script | Copy it to the EFI partition as | What it does |
|---|---|---|
| `scripts/a16-finish-boot.sh` | `\A16FIX.SH` | stage 1: `grub-install --no-nvram --removable` in the installed system, `update-grub`, log to the ESP as `A16BOOT.LOG` |
| `scripts/a16-stage-esp-boot.sh` | `\A16STAGE2.SH` | stage 2: diagnostics, plus a boot that needs nothing but the EFI partition |

Either copy them from Windows onto the ESP (it is a FAT volume Windows can mount), or — quicker —
do it in the live session before shutting down. **Rebooting into Windows is only needed for the
Windows-side copy helper**, which is the one piece not kept in this repository: the original was a
small PowerShell script (`a16-esp-stage2.ps1`) that mounted the EFI partition and copied the file,
so the same job can be done by hand:

    # Windows, elevated PowerShell: find the EFI system partition, give it a letter, copy, remove
    Get-Partition | Where-Object { $_.GptType -eq '{c12a7328-f81f-11d2-ba4b-00a0c93ec93b}' }
    mountvol S: /S
    Copy-Item .\A16FIX.SH S:\A16FIX.SH
    Copy-Item .\A16STAGE2.SH S:\A16STAGE2.SH
    mountvol S: /D

The repair scripts themselves are shell scripts and are unchanged; keep them on a stick so a future
installation does not need this step at all.

## Phase 4 — live session: finish the boot in two stages

Boot the stock daily again, open a terminal, and run the two stages in order. Confirm the installed root first (`findmnt -no SOURCE /`); it is not always the same
partition number.

**Stage 1**

    sudo mount /dev/nvme0n1p12 /mnt
    sudo bash /mnt/A16FIX.SH

It installs GRUB into the installed system with `--no-nvram --removable` (no `efibootmgr` needed),
generates `/boot/grub/grub.cfg`, and leaves `A16BOOT.LOG` at the root of the ESP. After this the ESP
carries the signed shim and GRUB and the installed system has its own menu — **but every entry fails
with**

    you need to load the kernel first

because this ESP's GRUB cannot read the kernel out of the installed root. That is what stage 2
removes from the path.

**Stage 2**

    sudo mount /dev/nvme0n1p12 /mnt
    sudo bash /mnt/A16STAGE2.SH

It writes, all on the ESP and all readable from Windows afterwards:

    A16DIAG2.TXT     kernel file format, /boot listing, module presence, ext4 features, fstab
    P17-GRUB.CFG     the installed system's generated menu
    A16STAGE2.LOG    the run's log
    A16ESP-BACKUP/   the configs that were in place before

and it stages a boot that needs **only the ESP**: the kernel, the initrd and the target's `arm64-efi`
GRUB module directory copied to `\a16boot\`, with a four-entry menu written to every location a GRUB
on this ESP reads a config from (`\EFI\ubuntu\grub.cfg`, `\EFI\ubuntu_snapdragon\grub.cfg`,
`\EFI\BOOT\grub.cfg`, `\boot\grub\grub.cfg`):

    0  installed Ubuntu, kernel + initrd from the ESP   (no ext4 involved)
    1  installed Ubuntu, its own generated menu         (keeps the normal path working)
    2  diagnostics                                      (prefix, cmdpath, module checks; 90 s)
    3  Windows Boot Manager

Reboot, press **Esc** at power-on for the boot options, choose the Linux entry and pick menu entry **0**. If it
fails, pick the diagnostics entry and photograph the panel — the diagnostics entry sleeps so the
screen can be read.

This is the menu the machine boots today.

## Phase 5 — first boot, then bring the machine up

On the installed system, in this order (details and verification commands per component are in
`docs/`):

1. **Check what the firmware gives you:**

       ls /sys/firmware/efi/efivars/
       sudo apt install -y efibootmgr && sudo efibootmgr -v

2. **Install the linux-next kernel and the device-tree boot entries** —
   `scripts/a16-install-next-kernel.sh`, then `scripts/a16-stage-dt-boot.sh`. The DT path with
   `acpi=off` is what gives internal input; ACPI mode cannot. See `docs/boot-options.md`.

3. **Firmware** — `scripts/a16-install-firmware.sh` (the DSP blobs staged in Phase 0),
   `scripts/a16-bt-setup.sh install` (Bluetooth firmware).

4. **Wi-Fi board data** — `scripts/make-a16-qcc2072-board-2.sh --install`. Without it the driver
   reports `failed to fetch board data` and never associates. See `docs/wifi.md`.

5. **Bluetooth** — `scripts/a16-bt-setup.sh`, then arm the patched device tree with
   `scripts/a16-bt-arm.sh` (the blob is `firmware/glymur-asus-zenbook-a16-ux3607oa-a16bt.dtb`, and
   the change is `patches/0001`). See `docs/bluetooth.md`.

6. **Display and GPU** — `docs/display-edp.md` and `docs/gpu-adreno.md`. Two things are needed: the
   Glymur GPU clock controller (`CONFIG_CLK_GLYMUR_GPUCC`, built as a module) so `msm` can bind at
   all, and the eDP link rate change in `patches/0009` so the panel trains. `patches/0006`/`0007` are
   the posted v8 PHY series, carried as well.

7. **Everything else** — input, battery, and what is still blocked (audio, suspend, external
   displays) in `docs/index.md`.

## Phase 6 — after any kernel change

The kernel modules built outside the stock kernel must match the running kernel exactly. A module built from
a tree whose config disagrees with the kernel's has **wrong struct offsets**, loads happily, and then
oopses in unrelated-looking places (cause: `pahole` missing → `CONFIG_DEBUG_INFO_BTF`
dropped → `SCHED_CLASS_EXT` dropped → `struct task_struct` shifted by 320 bytes).

    sudo bash scripts/a16-bootstrap.sh --check     # report tool/tree/patch/config/module state
    sudo bash scripts/a16-bootstrap.sh --all       # tools, tree, patches, config, build, install, options

`scripts/a16-fix-build-config.sh` does the config and verification on its own, and
`docs/build.md` explains the requirement and the checks (`thread_pid` must be 2144 on both sides).

## Not included in this repository

- **The Windows-side PowerShell helpers** (`a16-esp-stage2.ps1`, `a16-copy-harvest.ps1`). They lived
  on the Windows side of the machine. Phase 3 gives their `mountvol`-based equivalent for the one
  that matters; the harvest copier was `robocopy /J` plus `Get-FileHash`.
- **Firmware binaries.** They are proprietary and machine-specific; Phase 0 says exactly where each
  one comes from. The two device trees *are* included (`firmware/`), because they are small and the
  Bluetooth one is a working, reviewed change (`patches/0001`).
- **The earlier experimental attempts** (other distributions, ISO variants, dead ends). Only the path
  that worked is documented here.

## Repository layout

    README.md      this guide, in order
    steps/         one page per phase, with the commands and the things that go wrong
    scripts/       every script used, and scripts/README.md says when each one is used
    config/        kernel build inputs used for the installer media
    patches/       the changes this machine needs, with provenance
    docs/          per-component status and details for the finished system
    notes/         dated findings (audio, Bluetooth, Wi-Fi board data, display)
    firmware/      stock and Bluetooth-patched device trees
