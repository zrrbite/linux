# Yocto on the NUC

The [Yocto Project](https://www.yoctoproject.org) doesn't install a
distribution; it builds one from source, from recipes grouped in layers. You
choose what goes in, and you get a disk image (`.wic`) with a boot loader, a
kernel and a root filesystem. This guide builds a small console system with
SSH for the NUC10, writes it to a USB stick, and boots the NUC from the
stick. The build runs on the Windows workstation in WSL2; nothing is built on
the NUC. The NUC's own NVMe drive is never written: take the stick out and
the NUC boots what it booted before.

Checked against **Yocto Project 6.0.4 "Wrynose"** (released 2026-10-20) and
[its documentation](https://docs.yoctoproject.org/6.0.4/).

**Status:** written 2026-11-19, not yet tried on the NUC.

**Why 6.0:** on 2026-11-19 there are two current releases. 6.1 "Blacksail"
(October 2026) is the newest, but is supported only until April 2027. 6.0
"Wrynose" is the long-term support release, supported until April 2030, so
what you build now keeps getting fixes
([releases](https://www.yoctoproject.org/development/releases/)). Ubuntu
24.04 is on 6.0's list of supported build hosts. WSL2 is, in the docs' words,
"compatible but neither officially supported nor validated".

**Why `bitbake-setup`:** the combined `poky` repository has no branch for 6.0.
Its master branch stopped being updated with 5.3, and 6.0's
[quick start](https://docs.yoctoproject.org/6.0.4/brief-yoctoprojectqs/index.html)
uses `bitbake-setup`, which fetches BitBake, OpenEmbedded-Core and
`meta-yocto` as separate repositories. Older guides that begin with
`git clone .../poky` don't apply.

**Why not on the Mac:** Yocto doesn't build on macOS itself. For a non-Linux
host the docs point to a Linux container (Docker or Podman), which on an
Apple Silicon Mac runs in an arm64 Linux VM. Not confirmed whether 6.0
supports arm64 build hosts: the release ships aarch64 build tools and the
project's autobuilder has Arm-host builders, but the supported-distributions
list doesn't name CPU architectures.

---

## Before you start

- **The workstation.** WSL2 with Ubuntu 24.04, set up as in
  [wsl2-build-host.md](wsl2-build-host.md#1-install-wsl2-and-ubuntu), with the
  `.wslconfig` from [section 2](wsl2-build-host.md#2-memory-cpus-and-disk).
  The Yocto docs' minimum is 32 GB of RAM, measured on a 4-core machine. WSL
  gets 16 GB plus 16 GB of swap there, so step 4 limits the build to 10 jobs
  at once.
- **Disk.** The docs say 140 GB free for a build of `core-image-sato` (they
  measured 133 GB), and 40 GB at the peak if you add `rm_work`, which deletes
  each recipe's work files once it's built. This guide builds the smaller
  `core-image-base`, so those are upper bounds
  ([system requirements](https://docs.yoctoproject.org/6.0.4/ref-manual/system-requirements.html)).
- **Time.** The docs give no figure. The first build downloads several GB of
  sources and compiles a cross toolchain, the kernel and several hundred
  packages. Later builds reuse what's already built and take minutes.
- **A USB stick or USB SSD of 8 GB or more.** Writing the image erases it.
  The image's size isn't confirmed; it includes all of linux-firmware. Check
  the `.wic` file's size in step 5.
- **A network cable** to the NUC. This image has no Wi-Fi driver (see below).
- **An HDMI screen and a USB keyboard** for the first boot.

## Choices made here

**Machine: `genericx86-64`.** A Yocto build targets a `MACHINE`. Two fit a
Comet Lake NUC:

| | `genericx86-64` (used here) | `intel-corei7-64` |
|---|---|---|
| Layer | `meta-yocto-bsp`, already in the setup | [meta-intel](https://git.yoctoproject.org/meta-intel/tree/?h=wrynose), one more to clone. It has a `wrynose` branch (last commit 2026-10-02) and one maintainer, at Intel |
| Kernel | `linux-yocto` 6.18 | `linux-intel` 6.18, Intel's LTS tree |
| Ethernet (I219-V, `e1000e`), graphics (`i915`) | Yes | Yes |
| Wi-Fi and Bluetooth (AX201) | No: the kernel config has no `iwlwifi` or `btusb` | Yes |
| Boot loader | GRUB | systemd-boot |
| Booting from USB | Root found by PARTUUID, with `rootwait` | Root found by PARTUUID, but its systemd-boot `.wks` has no `rootwait` |
| CPU microcode updates | Not added | `intel-microcode` |
| Tested on | NUC5 and NUC6 ([README](https://git.yoctoproject.org/meta-yocto/tree/meta-yocto-bsp/README.hardware.md?h=wrynose)) | Tiger Lake and newer; Comet Lake isn't listed ([list](https://git.yoctoproject.org/meta-intel/tree/documentation/tested_hardware.md?h=wrynose)) |

`genericx86-64` needs no extra layer, and its image boots from a stick as it
is. The kernel's own docs describe `rootwait` as "useful for devices that
are detected asynchronously (e.g. USB and MMC devices)": without it, the
kernel can look for the root partition before the stick has shown up. The
cost is Wi-Fi, so this guide uses the Ethernet port. Adding the AX201's
drivers is one line in a layer of your own: see
[Next steps](#wi-fi-and-bluetooth).

**Image: `core-image-base`**, "a console-only image that fully supports the
target device hardware". `genericx86-64` adds the kernel modules and
linux-firmware through `MACHINE_EXTRA_RRECOMMENDS`, which `core-image-base`
installs and `core-image-minimal` doesn't. The minimal image would boot
without the `i915` module and without any firmware.

**Firmware.** In 6.0, linux-firmware is split into many packages, and the
package called `linux-firmware` is an empty one that pulls in all of them.
That covers what the NUC needs: `linux-firmware-i915` holds
`i915/kbl_dmc_ver1_04.bin`, the display firmware the kernel loads for Comet
Lake, and `linux-firmware-iwlwifi-misc` is the catch-all that holds the
AX201's files. It makes the image bigger than it has to be; trimming it is a
later job.

**Boot:** the machine's own `genericx86.wks.in` makes a GPT disk with an EFI
partition (GRUB and the kernel), an ext4 root partition and a small swap
partition. It boots by UEFI only, with Secure Boot off.

---

## 1. Install the host packages

In the Ubuntu 24.04 distro:

```bash
sudo apt-get update
sudo apt-get install build-essential chrpath cpio debianutils diffstat file \
  gawk gcc git iputils-ping libacl1 libcrypt-dev locales python3 python3-git \
  python3-jinja2 python3-pexpect python3-pip python3-subunit socat texinfo \
  unzip wget xz-utils zstd
sudo apt-get install python3-websockets
sudo apt-get install python3-venv whois
```

The first two `apt-get install` lines are the docs' list for Ubuntu
([Required Packages for the Build Host](https://docs.yoctoproject.org/6.0.4/ref-manual/system-requirements.html#required-packages-for-the-build-host)).
The third isn't in it: `python3-venv` because Ubuntu's Python can't make the
virtual environment in step 2 without it, and `whois` for `mkpasswd` in step
4.

BitBake also needs the `en_US.UTF-8` locale:

```bash
locale --all-locales | grep en_US.utf8
```

You should see `en_US.utf8`. If the line is empty, run
`sudo dpkg-reconfigure locales`, tick `en_US.UTF-8 UTF-8`, and run the check
again.

## 2. Fetch the layers with bitbake-setup

```bash
cd ~
python3 -m venv --clear ./bitbake-setup-venv
. ./bitbake-setup-venv/bin/activate
pip install bitbake-setup
bitbake-setup init --non-interactive poky-wrynose poky distro/poky machine/genericx86-64
```

The four words after `--non-interactive`:

- `poky-wrynose`: the Poky reference distribution on 6.0's branches. It
  follows the head of those branches, not the 6.0.4 tag, so you also get the
  fixes made since.
- `poky`: the plain build configuration. The alternative, `poky-with-sstate`,
  also downloads prebuilt pieces from the Yocto Project's mirror; the tool
  warns that it needs "a completely robust local network with sufficient
  bandwidth".
- `distro/poky`: Poky with SysVinit. (`distro/poky-altcfg` is the systemd
  variant.)
- `machine/genericx86-64`.

`bitbake-setup init` without arguments asks the same questions as numbered
menus.

It makes a top directory, `bitbake-builds`, in the folder you run it in:
`~/bitbake-builds`, with the setup in `poky-wrynose/` inside it. That keeps
the build under `~`, as it has to be
([why](wsl2-build-host.md#3-where-to-keep-build-trees)).

You should see, at the end:

```
To run builds, source the environment using
    . /home/<you>/bitbake-builds/poky-wrynose/build/init-build-env
```

## 3. Open the build environment

```bash
source ~/bitbake-builds/poky-wrynose/build/init-build-env
cd ~/bitbake-builds/poky-wrynose/build
bitbake-config-build list-fragments
```

You should see `machine/genericx86-64` and `distro/poky` among the enabled
fragments. Fragments are how this release switches settings on and off; the
two you chose at `init` set `MACHINE` and `DISTRO`.

Run the `source` line in every new terminal before you use `bitbake`. The
virtual environment from step 2 is only for `bitbake-setup` itself.

## 4. Configure the image

Your settings go at the end of `conf/local.conf` (in the `build` directory),
which is nearly empty to start with.

**A password.** The image gets a root password and a user of your own, both
from a hashed password. Make the hash:

```bash
printf '%q\n' "$(mkpasswd -m sha256crypt)"
```

Type the password at the prompt. Use only letters and digits: the NUC's
console has the US keyboard layout, so symbols aren't where the Danish
keyboard shows them. You should see a line like `\$5\$...\$...`; `printf
'%q'` puts the backslashes before each `$`, which the setting needs.

Then add this to the end of `conf/local.conf`, with your hash on the
`NUC_PASSWD` line. `nuc` is the user name; use another if you like.

```
# --- NUC10, see doc/yocto-guide.md ---

# An uncompressed .wic as well (the machine already makes .wic.zst and .wic.bmap)
IMAGE_FSTYPES += "wic"

# OpenSSH server
EXTRA_IMAGE_FEATURES += "ssh-server-openssh"

# Host name, time zone, and lspci for checking the hardware
hostname:pn-base-files = "nuc10-yocto"
DEFAULT_TIMEZONE = "Europe/Copenhagen"
IMAGE_INSTALL:append = " tzdata-core tzdata-europe pciutils"

# Fewer jobs at once: with WSL at 16 GB, 20 jobs can run out of memory
BB_NUMBER_THREADS = "10"
PARALLEL_MAKE = "-j 10"

# A root password and your own user, both with the password hashed above
IMAGE_CLASSES += "extrausers"
NUC_PASSWD = "\$5\$paste\$your-hash-here"
EXTRA_USERS_PARAMS = "\
    usermod -p '${NUC_PASSWD}' root; \
    useradd -m -p '${NUC_PASSWD}' nuc; \
"
```

What it gives you: root can log in at the NUC's console with the password.
Root can't log in over SSH with a password (the image's OpenSSH default), so
you SSH in as `nuc` and run `su -`. The user and password mechanism is the
[`extrausers`](https://docs.yoctoproject.org/6.0.4/ref-manual/classes.html#extrausers)
class. Don't put this file, or a layer with the hash in it, in a public
repository: a password hash can be cracked offline.

**Or, for a quick test,** skip the user lines and enable 6.0's ready-made
fragment instead:

```bash
bitbake-config-build enable-fragment core/yocto/root-login-with-empty-password
```

It adds the image features `allow-empty-password`, `empty-root-password` and
`allow-root-login`: root logs in with no password, at the console and over
SSH. **Anyone on your network can then log in to the NUC as root.** These
three features are what `debug-tweaks` used to do; `debug-tweaks` was
removed in 5.2, so older guides that use it don't build.

**The keyboard stays US.** A Danish console keymap isn't a simple setting in
Yocto: the image's keymap script only loads a prebuilt map file named after
the kernel version. For one session, add `kbd kbd-keymaps` to the
`IMAGE_INSTALL:append` line and run `loadkeys dk` after logging in at the
console.

## 5. Build

```bash
bitbake core-image-base
```

Expect this warning at the start of every build; it's about the disk space
WSL doesn't give back by itself
([section 5](wsl2-build-host.md#5-get-the-disk-space-back)):

```
WARNING: You are running bitbake under WSLv2, this works properly but you should optimize your VHDX file eventually to avoid running out of storage space
```

You should see, at the end:

```
NOTE: Tasks Summary: Attempted ... tasks of which ... didn't need to be rerun and all succeeded.
```

Then check the image:

```bash
ls -lL tmp/deploy/images/genericx86-64/core-image-base-genericx86-64.rootfs.wic*
```

You should see three files ending in `.wic`, `.wic.bmap` and `.wic.zst`.
They're links to the newest build; each build also leaves a copy with a
timestamp in its name. The `.wic` size is what the stick has to hold.

**If a job dies with `Killed`,** WSL ran out of memory. Lower both numbers
in the "Fewer jobs" lines from step 4, to 8 or 6, and run `bitbake` again;
it picks up where it stopped. Don't fix it by giving WSL more memory: at
`memory=24GB` the build here starved Windows instead
([section 2](wsl2-build-host.md#2-memory-cpus-and-disk)).

**`User namespaces are not usable by BitBake, possibly due to AppArmor`**
shouldn't happen here. BitBake uses user namespaces to cut build tasks off
from the network, and on a normal Ubuntu 24.04 install AppArmor blocks them.
That block is a setting in Ubuntu's own kernel, which WSL doesn't run.
Checked on the workstation on 2026-11-19: WSL's kernel (6.6.87.2) doesn't
have the setting, and an ordinary user could create a user and network
namespace (`unshare --user --map-root-user --net`).

**To save disk space,** add `INHERIT += "rm_work"` to `conf/local.conf`.
For the docs' `core-image-sato` it cuts the final size from 140 GB to 22 GB
([details](https://docs.yoctoproject.org/6.0.4/dev-manual/disk-space.html)).

Where everything lands:

| What | Where |
|---|---|
| Your settings | `~/bitbake-builds/poky-wrynose/build/conf/local.conf` |
| The layers | `~/bitbake-builds/poky-wrynose/layers/` |
| Downloaded sources | `~/bitbake-builds/.bitbake-setup-downloads/` |
| Shared-state cache (built pieces, reused) | `~/bitbake-builds/.sstate-cache/` |
| Work files, the biggest part | `~/bitbake-builds/poky-wrynose/build/tmp/` |
| Images | `build/tmp/deploy/images/genericx86-64/` |

The downloads and the cache are shared by every setup under
`~/bitbake-builds`. `bitbake-getvar --value DL_DIR` (or `SSTATE_DIR`) shows
the paths if you want to check. You can delete `build/tmp` at any time; the
next build puts it back from the cache.

## 6. Optional: boot it in QEMU first

This boots the same `.wic` in a virtual machine inside WSL, to check the
login and SSH before you write a stick. Not confirmed for `genericx86-64`:
the Yocto docs show `runqemu` with the `qemu*` machines; the commands below
use `runqemu`'s documented options and come from reading its source.

KVM makes it fast, and WSL2 has it when nested virtualization is on (the
default on Windows 11):

```bash
ls -l /dev/kvm
```

The shared page found it present on the workstation. If `runqemu` says you
have no permission on it, run `sudo usermod -aG kvm $USER`, then
`wsl --shutdown` in PowerShell, and open the distro again.

The genericx86-64 image only boots by UEFI, so QEMU needs UEFI firmware.
Build it once, then start the VM:

```bash
bitbake ovmf
runqemu genericx86-64 core-image-base wic ovmf nographic slirp snapshot kvm
```

- `wic ovmf`: boot the `.wic` with the OVMF UEFI firmware.
- `nographic`: the VM's serial console is your terminal; no window.
- `slirp`: user-mode networking, no `sudo`. The VM's SSH port appears as
  port 2222 on the WSL side.
- `snapshot`: throw away what the VM writes, so the `.wic` stays as built.
- `kvm`: leave it out if `/dev/kvm` doesn't exist.

You should see the GRUB menu, kernel messages, a pause while the start
scripts run on the VM's (invisible) screen, then `nuc10-yocto login:`. From
a second WSL terminal:

```bash
ssh -p 2222 nuc@127.0.0.1
```

To stop the VM, run `poweroff` in it, or press `Ctrl+A` then `X`.

## 7. Write the stick

Copy the image to Windows, with an `.img` name so Rufus lists it:

```bash
cp tmp/deploy/images/genericx86-64/core-image-base-genericx86-64.rootfs.wic \
  /mnt/c/Users/<you>/Downloads/nuc10-yocto.img
```

Then write it with Rufus or balenaEtcher, as in
[wsl2-build-host.md](wsl2-build-host.md#4-write-an-image-to-a-usb-stick).
**Writing erases the whole stick; check you picked the right drive.**

**From a Linux machine,** the Yocto docs recommend `bmaptool`, which skips
the empty parts of the image and is much faster than `dd`. Copy the `.wic`
and its `.wic.bmap` there, side by side, and run
([docs](https://docs.yoctoproject.org/6.0.4/dev-manual/bmaptool.html)):

```bash
sudo bmaptool copy core-image-base-genericx86-64.rootfs.wic /dev/sdX
```

`/dev/sdX` is the stick; **check it with `lsblk` first**, because `bmaptool`
overwrites whichever disk you name.

## 8. Boot the NUC

Plug in the stick, and leave other USB drives out (see step 9). Power on and
press **F2** for the BIOS setup: **Secure Boot off**, because the image's
GRUB and kernel aren't signed. Save and exit, then press **F10** on the way
up and pick the USB stick. Details are in
[nuc10.md](nuc10.md#bios-settings) and
[nuc10.md](nuc10.md#booting-from-usb).

You should see a GRUB menu with one entry, `boot`, for five seconds, then
kernel messages and maybe a Yocto splash screen, and then:

```
Poky (Yocto Project Reference Distro) 6.0.4 nuc10-yocto tty1

nuc10-yocto login:
```

The version may be a later 6.0.x, because the setup follows the `wrynose`
branches. Log in as `root` with your password (or with no password, if you
used the quick-test fragment).

**If it stops at `Waiting for root device PARTUUID=...`,** the kernel
doesn't see the stick. Try another USB port, or another stick.

## 9. First checks

```bash
uname -a                         # Linux nuc10-yocto 6.18.x-yocto-standard ...
cat /proc/cmdline                # root=PARTUUID=... rootwait ...
ip a                             # eth0 with an inet address, if the cable is in
dmesg | grep -i -e firmware -e dmc
lspci -k                         # each device, and the driver it uses
```

You should see `Finished loading DMC firmware i915/kbl_dmc_ver1_04.bin`, and
no lines saying a firmware file failed to load. In `lspci -k`, the Ethernet
controller uses `e1000e` and the graphics uses `i915`; the Wi-Fi controller
has no driver, as expected.

**SSH in** from the workstation (WSL or PowerShell both have `ssh`), with the
address from `ip a`:

```bash
ssh nuc@<nuc-ip>
su -
```

The image also runs Avahi, so `ssh nuc@nuc10-yocto.local` may work from the
Mac. Not confirmed from Windows or WSL.

**About `/etc/fstab`:** root is found by PARTUUID, but the image's
`/etc/fstab` names the boot and swap partitions `/dev/sda1` and `/dev/sda3`.
On this NUC the stick is normally the only `sd` disk, so that matches. With
another USB drive plugged in first, those two may not mount; the root
filesystem isn't affected.

To shut down: `poweroff`.

---

## Changing the image and rebuilding

Edit `conf/local.conf` and run `bitbake core-image-base` again. BitBake works
out what the change affects and reuses everything else from the
shared-state cache. Adding a user or a package that's already built only
redoes the root filesystem and the image: minutes, not hours. Then write the
new `.wic` to the stick as in step 7.

Each build leaves a timestamped copy of the image in
`tmp/deploy/images/genericx86-64/`. Delete the old ones now and then, and
compact WSL's disk afterwards
([section 5](wsl2-build-host.md#5-get-the-disk-space-back)).

**To pick up later 6.0 fixes,** update the layers and rebuild. Run it with
the build environment sourced and the venv from step 2 active:

```bash
bitbake-setup update --update-bb-conf no
```

`--update-bb-conf no` leaves `local.conf` as it is.

## Next steps

### Your own layer

Settings in `local.conf` are fine to start with. A layer holds them as
recipes you can keep in git:

```bash
bitbake-layers create-layer ../layers/meta-nuc10
bitbake-layers add-layer ../layers/meta-nuc10
```

You should see `meta-nuc10` in `bitbake-layers show-layers`. The layer
starts with an example recipe; see
[Creating Your Own Layer](https://docs.yoctoproject.org/6.0.4/dev-manual/layers.html).
Later, an image recipe of your own in it can take over the lines from
`local.conf`.

### Wi-Fi and Bluetooth

The `genericx86-64` kernel leaves out the AX201's drivers. The kernel
recipe's `KERNEL_FEATURES` adds them from the Yocto kernel's own config
fragments
([docs](https://docs.yoctoproject.org/6.0.4/kernel-dev/advanced.html)). In
your layer:

```bash
mkdir -p ../layers/meta-nuc10/recipes-kernel/linux
cat > ../layers/meta-nuc10/recipes-kernel/linux/linux-yocto_%.bbappend <<'EOF'
KERNEL_FEATURES:append:genericx86-64 = " features/iwlwifi/iwlwifi.scc features/bluetooth/bluetooth-usb.scc"
EOF
bitbake core-image-base
```

This rebuilds the kernel. The firmware is already in the image, and so are
`wpa_supplicant` and `iw`. After booting the new image, you should see
`iwlwifi` load a firmware file in `dmesg | grep iwlwifi` (not confirmed: on
Comet Lake it should be an `iwlwifi-QuZ-a0-hr-b0-*.ucode`) and a `wlan0` in
`ip a`. The image's `/etc/network/interfaces` already has a `wlan0` entry
that reads `/etc/wpa_supplicant.conf`; connecting to a network is untried
here.

### devtool

`devtool` is the tool for working on a recipe's source
([docs](https://docs.yoctoproject.org/6.0.4/dev-manual/devtool.html)):

- `devtool add <name> <source-url>` makes a new recipe from a source tree.
- `devtool modify <recipe>` checks out an existing recipe's source to change.
- `devtool build <recipe>` builds it.
- `devtool deploy-target <recipe> root@<nuc-ip>` copies the result onto the
  running NUC over SSH. With the password setup from step 4 that needs your
  SSH key in `/root/.ssh/authorized_keys` on the NUC, since root can't log
  in over SSH with a password.
- `devtool finish <recipe> ../layers/meta-nuc10` turns your changes into a
  recipe or patches in your layer.
