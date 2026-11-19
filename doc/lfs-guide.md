# Linux From Scratch on the NUC

[Linux From Scratch](https://www.linuxfromscratch.org/lfs/) (LFS) is a book
that has you build a Linux system from source, one package at a time. You do
it to learn what every piece of a Linux system does and how the pieces fit
together. What you end up with is a bare console system, not a daily driver.

This guide wraps the book for this setup. You build in WSL2 on the Windows
workstation, into a disk image file. Then you write the image to a USB stick
and boot the NUC10 from it. The NUC's own SSD is never written. The book is
still the step-by-step. This guide says which book to use, what to do
differently in each chapter, and how to do the parts the book leaves to you:
the disk (here, an image file), the kernel config for the NUC, and booting
from USB with UEFI.

Checked against **LFS 13.1-systemd** (released 2026-09-01) and
[the online book](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/),
with [BLFS 13.1-systemd](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/)
for the extras. The book builds Linux 7.1.8, GCC 16.2.0, glibc 2.44 and
systemd 261.2. It has no [errata](https://www.linuxfromscratch.org/lfs/errata/13.1-systemd/)
as of 2026-11-19.

**Status:** written 2026-11-19. Sections 1 and 2 and the chapter 3
downloads were done on the workstation that day; the build itself (chapter 4
on) hasn't been started. Not yet tried on the NUC.

**Why the systemd edition:** the System V edition is no longer updated. The
LFS release notes for 13.0 and 13.1 say it stays at 12.4
([news](https://www.linuxfromscratch.org/news.html)). systemd also matches
the Arch NUCs, so `systemctl`, `journalctl` and systemd-networkd work the way
you already know.

**Why not on the Mac:** the book needs a Linux host, and it builds for the
host's own CPU (`LFS_TGT=$(uname -m)-lfs-linux-gnu`). macOS isn't Linux. A
Linux VM on an Apple Silicon Mac runs on ARM, so it would build a system the
NUC's x86_64 CPU can't run.

---

## Before you start

- **Time.** By the book's own figures, compiling and testing add up to about
  120 SBU. An SBU is the time binutils pass 1 takes on one core of your
  machine ([4.5](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter04/aboutsbus.html)).
  GCC with its test suite is the biggest single item, at 53 SBU. On the
  i7-12700K that's hours of machine time, not days. Reading, typing and
  checking each step is what stretches it over several evenings. The book
  is written to be done in stages
  ([2.3](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter02/stages.html)).
- **The workstation.** You need WSL2 with Ubuntu 24.04, set up as in
  [wsl2-build-host.md](wsl2-build-host.md#1-install-wsl2-and-ubuntu). The
  book recommends at least 4 cores and 8 GB of RAM. The `.wslconfig` in
  [section 2](wsl2-build-host.md#2-memory-cpus-and-disk) gives WSL far more
  than that.
- **Disk.** About 40 GB free inside WSL: the image itself (20 GiB), the
  sources, and a backup. You also need 20 GB on C: for the copy you write to
  the stick.
- **A USB stick of 32 GB or more**, USB 3. The image is 20 GiB, and a "32 GB"
  stick holds about 29.8 GiB. If you'll use the system much, get a small USB
  SSD instead. It's much faster and wears better. Writing the image erases
  whatever drive you write it to.
- **A network cable** to the NUC for the first boot. Wi-Fi needs packages
  from BLFS.
- A monitor on HDMI and a USB keyboard.

How this differs from the book, chapter by chapter:

| Book | What's different here |
|---|---|
| 2.2 Host requirements | Ubuntu needs a few fixes ([section 1](#1-prepare-ubuntu)) |
| 2.4 to 2.7 Partition, format, mount | An image file on a loop device ([section 2](#2-the-disk-an-image-file)) |
| 3 to 7 | As written, plus a few WSL notes ([section 3](#3-chapters-3-to-7)) |
| 8 | Time zone and locale; skip GRUB ([section 4](#4-chapter-8)) |
| 9 | Wired DHCP, Danish keymap ([section 5](#5-chapter-9)) |
| 10 | `fstab` by PARTUUID, the NUC's kernel config, firmware, and the kernel's EFI stub instead of GRUB ([section 6](#6-chapter-10-fstab-kernel-firmware-boot)) |
| 11 | Unmount, write the stick, boot the NUC ([sections 7 to 9](#7-chapter-11-and-writing-the-stick)) |

---

## 1. Prepare Ubuntu

Book: [2.2 Host System Requirements](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter02/hostreqs.html).

In the Ubuntu 24.04 terminal:

```bash
sudo apt update
sudo apt install build-essential bison gawk m4 texinfo python3 wget dosfstools fdisk
```

`build-essential` brings gcc, g++, make and patch. bison also installs
`yacc`, and installing gawk makes `awk` point to it instead of Ubuntu's mawk.
`texinfo` gives `texi2any`. `dosfstools` formats the EFI system partition,
and `fdisk` has `sfdisk`, which partitions the image.

Ubuntu points `/bin/sh` at dash, and the book needs bash. `dpkg-reconfigure
dash` doesn't change it any more (it stopped working in Ubuntu 22.10:
[Launchpad #1996243](https://bugs.launchpad.net/ubuntu/+source/dash/+bug/1996243)),
so change the link by hand:

```bash
sudo ln -sfv bash /bin/sh
```

An update to the dash package can put the link back. The version check
catches that, so run it again whenever you come back after an update.

Now create and run the book's `version-check.sh` from 2.2. You should see
only `OK` lines, ending with `OK: nproc reports 20 logical cores are
available`. Before the fixes above, you'd get `ERROR` lines for `sh`, `awk`
and `yacc`. Ubuntu 24.04's GCC 13 and binutils 2.42 are older than the
newest versions the book has tested (GCC 16.2.0, binutils 2.47), and that's
the safe side to be on. WSL's kernel is far newer than the 5.10 the book
requires.

---

## 2. The disk: an image file

Book: replaces [2.4](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter02/creatingpartition.html)
to [2.7](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter02/mounting.html).

The book assumes you have a spare partition. Here the disk is a 20 GiB file
in your WSL home, partitioned the way the stick will be: a 512 MiB EFI system
partition (ESP) and an ext4 root. A loop device makes Linux treat the file
as a disk. 20 GiB covers the book's needs (2.4 asks for 10 GB, or 30 GB with
room to grow) plus the kernel tree and a few BLFS packages, and it fits on a
32 GB stick. The file is sparse, so it only uses as much space as has been
written to it.

The book runs most host-side steps as root, with `LFS` set
([2.6](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter02/aboutlfs.html)).
Open a root shell and put the settings in root's profile, so every root
shell has them. `IMG` is this guide's own variable, the path of the image
file. Replace `<you>` with your WSL user name:

```bash
sudo -i
cat >> /root/.profile << "EOF"
export LFS=/mnt/lfs
export IMG=/home/<you>/lfs/lfs.img
umask 022
EOF
source /root/.profile
```

Create the image and partition it:

```bash
mkdir -pv $(dirname $IMG)
truncate -s 20G $IMG
sfdisk $IMG << "EOF"
label: gpt
size=512MiB, type=uefi, name="LFS ESP"
type=linux, name="LFS root"
EOF
```

You should see the new table, with a 512M "EFI System" partition and a
"Linux filesystem" partition, followed by `The partition table has been
altered.`

Attach it as a loop device. `-P` makes the kernel read the partition table
and create a device for each partition:

```bash
LOOP=$(losetup -fP --show $IMG)
echo $LOOP
lsblk $LOOP
```

You should see something like `/dev/loop0`, and `lsblk` should list
`loop0p1` (512M) and `loop0p2` (19.5G). If the two partition lines are
missing, run `partx -a $LOOP`.

**Before you format: check that `echo $LOOP` printed a `/dev/loopN`.** In
WSL, `/dev/sda`, `/dev/sdb` and so on are the virtual disks that hold WSL
and your Ubuntu. Formatting one of them destroys the distro.

```bash
mkfs.fat -F 32 -n LFSESP ${LOOP}p1
mkfs -v -t ext4 -L lfs ${LOOP}p2
```

You'll use the labels `LFSESP` and `lfs` later to find the partitions without
needing the loop device's number. Mount root as the book does in 2.7:

```bash
mkdir -pv $LFS
mount -v -t ext4 ${LOOP}p2 $LFS
chown root:root $LFS
chmod 755 $LFS
```

You should see `findmnt $LFS` print a line with `/dev/loop0p2 ext4`.

### Coming back after WSL restarts

WSL stops its VM a little while after you close the last terminal. `wsl
--shutdown` and a Windows reboot stop it too. The loop device and every mount
go away when it stops, and so does any build running in that terminal. Keep
the terminal open during long builds, or run them in `tmux`, and don't let
Windows go to sleep. To pick up where you left off:

```bash
sudo -i
LOOP=$(losetup -j $IMG | cut -d: -f1)
[ -n "$LOOP" ] || LOOP=$(losetup -fP --show $IMG)
mountpoint -q $LFS || mount -v -t ext4 ${LOOP}p2 $LFS
```

From chapter 7 on, also remount the virtual file systems
([7.3](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter07/kernfs.html),
or the `mount-virt.sh` script in
[11.5.2.1](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter11/afterlfs.html))
and enter the chroot again
([7.4](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter07/chroot.html)).

---

## 3. Chapters 3 to 7

Follow the book. The notes below are the only differences.

- **[3. Packages and Patches](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter03/introduction.html):**
  download into `$LFS/sources` from the Ubuntu side, as root. Use the
  systemd list,
  [`wget-list-systemd`](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/wget-list-systemd),
  and check the downloads with
  [`md5sums`](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/md5sums).
  On 2026-11-19 all 94 packages downloaded and passed `md5sum -c`: 628 MB.
- **[4.4 Setting Up the Environment](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter04/settingenvironment.html):**
  Ubuntu has an `/etc/bash.bashrc`. Move it out of the way as the book says,
  and put it back at the start of chapter 7. `MAKEFLAGS=-j$(nproc)` uses all
  20 threads. If a build dies with `Killed signal terminated program
  cc1plus`, WSL ran out of memory. Lower the `-j` number, or give WSL more
  memory.
- **[4.5 About SBUs](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter04/aboutsbus.html):**
  time binutils pass 1 as the book suggests. That gives you your SBU, and
  from it how long the rest will take.
- **Chapters 5 and 6:** nothing WSL-specific. If `su - lfs` hangs without
  showing a prompt, type `fg`, as the note in 4.3 says.
- **[7.3 Virtual Kernel File Systems](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter07/kernfs.html):**
  the bind mount of `/dev` and the `devpts`, `proc`, `sysfs` and `tmpfs`
  mounts are plain Linux and work the same in WSL2. The book's `/dev/shm`
  test handles either way WSL sets up `/dev/shm`.
- **[7.15 Cleaning up and Saving](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter07/cleanup.html):**
  make the backup. If chapter 8 goes wrong, it saves you redoing chapters 5
  to 7. Keep it under `/root` (inside WSL), not on `/mnt/c`.

Not confirmed: I found no first-hand account of a complete LFS 13.x build in
WSL2. The few reports I found of LFS problems under WSL were ordinary book
mistakes, not WSL problems: a missing `$LFS/lib` symlink from 4.2, which
makes `chroot` fail with `/usr/bin/env: No such file or directory`, and
virtual file systems not remounted after a restart.

What this guide relies on is checked. WSL's kernel has loop devices, GPT
partitions, devtmpfs, ext4, vfat and Unix98 PTYs built in (in the config of
`linux-msft-wsl-6.18.58.1`, and of 6.6.87.2, which the workstation's WSL
2.6.3 runs). On 2026-11-19, sections 1 and 2 and the chapter 3 downloads
were run on the workstation, in Ubuntu 24.04: `version-check.sh` printed only
`OK` lines; `sfdisk` took the `type=uefi` names; `losetup -P` created
`loop0p1` and `loop0p2`; formatting and mounting `$LFS` worked. A separate
test with a small image also checked `blkid -L`, the PARTUUID lookup and
`fstrim`.

---

## 4. Chapter 8

Book: [8. Installing Basic System Software](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter08/introduction.html).

- **Test failures.** Compare each one with the list on the package's page.
  The tests run against WSL's kernel, not the one the NUC will boot, and the
  systemd page itself says some tests fail depending on kernel options. If a
  test fails that the book doesn't mention, check whether it involves the
  kernel before you rebuild the package.
- **[8.5 Glibc](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter08/glibc.html),
  time zone:**
  ```bash
  ln -sfv /usr/share/zoneinfo/Europe/Copenhagen /etc/localtime
  ```
  Optionally, add `localedef -i en_DK -f UTF-8 en_DK.UTF-8` to the book's
  list of locales. It gives English messages with Danish date and number
  formats.
- **[8.65 GRUB](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter08/grub.html):**
  skip it. This guide boots the kernel without a boot loader
  ([section 6](#boot-the-kernels-efi-stub)). If you want to try GRUB later,
  build only the 64-bit UEFI part, 8.65.2.
- **[8.77 Systemd](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter08/systemd.html):**
  as written.

---

## 5. Chapter 9

Book: [9. System Configuration](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter09/introduction.html).

**[9.2 Network](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter09/network.html):**
use DHCP on the wired port (the NUC's I219-V):

```bash
cat > /etc/systemd/network/10-wired.network << "EOF"
[Match]
Name=en*

[Network]
DHCP=ipv4

[DHCPv4]
UseDomains=true
EOF
echo lfs > /etc/hostname
```

`en*` matches any wired interface, so you don't need to know the name udev
will give the port. You'll see the name in `ip link` after boot. Don't
create `/etc/resolv.conf`: systemd-resolved creates it at boot. In
`/etc/hosts`, leave out the line with an IP address, as the book allows when
you use DHCP.

**[9.5 Clock](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter09/clock.html):**
Arch keeps the NUC's hardware clock in UTC, so don't create `/etc/adjtime`.
Without it, systemd assumes UTC.

**[9.6 Console](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter09/console.html):**
Danish keyboard:

```bash
cat > /etc/vconsole.conf << "EOF"
KEYMAP=dk
FONT=Lat2-Terminus16
EOF
```

**[9.7 Locale](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter09/locale.html):**
`LANG=en_DK.UTF-8` if you created that locale in 8.5. Otherwise use the
book's example.

---

## 6. Chapter 10: fstab, kernel, firmware, boot

This is where the NUC comes in. Stay in the chroot unless a step says
otherwise.

### Find the partition IDs

```bash
ROOT_PARTUUID=$(blkid -s PARTUUID -o value $(blkid -L lfs))
ESP_PARTUUID=$(blkid -s PARTUUID -o value $(blkid -L LFSESP))
echo "root: $ROOT_PARTUUID  esp: $ESP_PARTUUID"
```

You should see two different UUIDs. If either is empty, the image isn't
attached. These are the GPT partition IDs. They go onto the stick
unchanged, so the NUC finds root whatever `/dev/sdX` name the stick gets,
next to its own NVMe disk. Use the PARTUUID, not the filesystem UUID. The
kernel understands `root=PARTUUID=` on its own, while `root=UUID=` needs an
initramfs (the note in
[10.4.5](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter10/grub.html)).
These are shell variables, so if you leave the chroot, run the commands
again. If you ever recreate the image, the IDs change, and you have to
update `fstab` and the kernel command line.

### fstab

Book: [10.2](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter10/fstab.html).
Here `EOF` has no quotes, so the shell fills in the variables:

```bash
cat > /etc/fstab << EOF
# Begin /etc/fstab

# file system            mount-point  type  options                                                    dump  fsck
PARTUUID=$ROOT_PARTUUID  /            ext4  defaults                                                   1     1
PARTUUID=$ESP_PARTUUID   /boot/efi    vfat  noauto,rw,relatime,codepage=437,iocharset=iso8859-1         0     0

# End /etc/fstab
EOF
cat /etc/fstab
```

You should see the two UUIDs in the file. The ESP is only needed when you
install a new kernel. LFS also has no `fsck.vfat`, which comes with
dosfstools in BLFS. `noauto` and fsck order `0` keep the boot from depending
on the ESP. There's no swap: a swap partition on a USB stick wears it out,
and the NUC has enough RAM for a console system.

### Kernel config

Book: [10.3 Linux-7.1.8](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter10/kernel.html).
Follow the book first: `make mrproper`, `make defconfig`, then `make
menuconfig` with every setting on the book's list, including its "booting
with UEFI" part. Then, in the kernel source directory, add what the NUC
needs:

```bash
scripts/config --disable WERROR \
  --enable EFI --enable EFI_STUB --enable EFIVAR_FS \
  --enable VFAT_FS --enable NLS_CODEPAGE_437 --enable NLS_ISO8859_1 \
  --enable USB_XHCI_HCD --enable USB_STORAGE --enable USB_UAS \
  --enable BLK_DEV_SD --enable EXT4_FS --enable BLK_DEV_NVME \
  --enable E1000E \
  --module DRM_I915 \
  --module CFG80211 --module MAC80211 --module IWLWIFI --module IWLMVM \
  --enable CMDLINE_BOOL \
  --set-str CMDLINE "root=PARTUUID=$ROOT_PARTUUID rootwait ro"
make olddefconfig
```

| Option | Set to | Why |
|---|---|---|
| `WERROR` | off | `make defconfig` turns it on; the book wants it off |
| `EFI`, `EFI_STUB`, `EFIVAR_FS` | built in | The firmware starts the kernel directly ([below](#boot-the-kernels-efi-stub)) |
| `VFAT_FS`, `NLS_CODEPAGE_437`, `NLS_ISO8859_1` | built in | To mount the ESP, with the codepages `fstab` names |
| `USB_XHCI_HCD`, `USB_STORAGE`, `USB_UAS`, `BLK_DEV_SD` | built in | Root is on USB. With no initramfs, everything needed to reach it must be in the kernel. UAS is what USB SSDs use, and USB disks appear as SCSI disks (`sd`) |
| `EXT4_FS` | built in | The root filesystem |
| `BLK_DEV_NVME` | built in | Lets LFS see the NUC's own SSD. Not needed to boot |
| `E1000E` | built in | The I219-V Ethernet port. Needs no firmware |
| `DRM_I915` | module | Intel UHD Graphics. Loads firmware from disk, so it has to start after root is mounted |
| `CFG80211`, `MAC80211`, `IWLWIFI`, `IWLMVM` | module | The AX201 Wi-Fi. Loads firmware from disk, as above |
| `CMDLINE_BOOL`, `CMDLINE` | on, with root | The kernel command line, built into the kernel, since there's no boot loader to pass one. `rootwait` makes the kernel wait for the USB disk to appear |

`scripts/config` doesn't check dependencies, and `make olddefconfig` quietly
drops what it can't satisfy, so check the result:

```bash
grep -E '^CONFIG_(EFI_STUB|USB_XHCI_HCD|USB_XHCI_PCI|USB_STORAGE|USB_UAS|BLK_DEV_SD|EXT4_FS|VFAT_FS|BLK_DEV_NVME|E1000E|DRM_I915|IWLWIFI|IWLMVM|CMDLINE)=' .config
```

You should see `=y` on every line except `DRM_I915`, `IWLWIFI` and
`IWLMVM`, which show `=m`. The last line should read
`CONFIG_CMDLINE="root=PARTUUID=<your root UUID> rootwait ro"`.

Then build and install as the book says: `make`, `make modules_install`, and
the copies to `/boot`. `make modules_install` matters here, because the
graphics and Wi-Fi drivers are modules.

### Firmware

LFS has no `wget`, so get the two firmware files from the Ubuntu side. In a
second WSL terminal, as root (`sudo -i`):

```bash
FW=https://git.kernel.org/pub/scm/linux/kernel/git/firmware/linux-firmware.git/plain
wget -P $LFS/usr/lib/firmware      $FW/intel/iwlwifi/iwlwifi-QuZ-a0-hr-b0-77.ucode
wget -P $LFS/usr/lib/firmware/i915 $FW/i915/kbl_dmc_ver1_04.bin
```

`/lib` is a link to `usr/lib` in LFS, so this is the `/lib/firmware` the
kernel reads. Drivers load firmware from there when they start, which only
works once root is mounted. That's why i915 and iwlwifi are modules, which
is also what BLFS's
[About Firmware](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/postlfs/firmware.html)
recommends.

Where the file names come from, checked in the Linux 7.1 source and the
linux-firmware repository on 2026-11-19:

- **Wi-Fi.** The AX201 on Comet Lake reports itself as a "QuZ" device. For
  that device the driver accepts exactly firmware API 77, so it needs
  `iwlwifi-QuZ-a0-hr-b0-77.ucode`. In linux-firmware the file lives in
  `intel/iwlwifi/`, with a link at the top level that's listed in the repo's
  `WHENCE` file. The kernel asks for the top-level name, and that's where
  `wget -P` puts it. The card is an AX201 soldered to the board, per Intel's
  [technical product specification](https://dlcdnets.asus.com/pub/ASUS/NUC/TPS/NUC10i357FN_TechProdSpec.pdf)
  (Rev 003). If the driver asks for a different file anyway, section 9 shows
  how to see which.
- **Graphics.** On Comet Lake, i915 loads the Kaby Lake display firmware,
  `i915/kbl_dmc_ver1_04.bin`. It doesn't load GuC or HuC firmware on this
  generation.

The other way to do it is to clone the whole repository and run
`./copy-firmware.sh $LFS/usr/lib/firmware`, which copies every file and
creates the links. If you'd rather build i915 and iwlwifi into the kernel,
list the files in `CONFIG_EXTRA_FIRMWARE` (see "Include Firmware Blobs in
the Kernel Image" on the BLFS page).

### Boot: the kernel's EFI stub

With `EFI_STUB`, the kernel image is itself a UEFI program
([kernel docs](https://docs.kernel.org/admin-guide/efi-stub.html)). On a
removable disk, UEFI firmware looks for `EFI/BOOT/BOOTX64.EFI` on a FAT
partition. Copy the kernel there and the NUC boots it with no boot loader,
using the command line built into the kernel:

```bash
mkdir -pv /boot/efi
mount -v -t vfat $(blkid -L LFSESP) /boot/efi
mkdir -pv /boot/efi/EFI/BOOT
cp -v arch/x86/boot/bzImage /boot/efi/EFI/BOOT/BOOTX64.EFI
ls -l /boot/efi/EFI/BOOT
umount -v /boot/efi
```

You should see `BOOTX64.EFI`, the same size as `arch/x86/boot/bzImage`.

**Why the stub and not GRUB:** LFS 13.1 now covers UEFI GRUB in the book
itself (8.65.2, and `grub-install --target=x86_64-efi --removable` in
[10.4.4.2](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter10/grub.html)),
so you no longer need BLFS for it. It gives you a boot menu where you can
edit the command line. But `grub-install` has to work out which disk `/boot`
is on, and it has been reported to fail on loop-device partitions with
`cannot find a GRUB drive for /dev/loop0p1`
([grub-devel, 2020](https://lists.libreplanet.org/archive/html/grub-devel/2020-06/msg00072.html)),
which is exactly where the image lives during the build. Not confirmed
whether GRUB 2.14 still does this. The stub has no installer that can go
wrong. The cost is that changing the command line means rebuilding the
kernel. If you want GRUB later, build it and run `grub-install` on the NUC
itself, where the stick is a real disk. It replaces `BOOTX64.EFI`.

---

## 7. Chapter 11 and writing the stick

Book: [11.1](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter11/theend.html)
and [11.3](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter11/reboot.html).

1. Create the release files as in 11.1, and set a root password
   (`passwd root`). You log in with it on the NUC.
2. If you want to log in over SSH from the start, build OpenSSH now
   ([section 10](#10-next-steps)).
3. Leave the chroot and unmount the virtual file systems as 11.3 shows.
   Then, instead of rebooting, unmount and detach the image:
   ```bash
   fstrim -v $LFS
   umount -v $LFS
   losetup -d $(losetup -j $IMG | cut -d: -f1)
   losetup -j $IMG
   ```
   The last command should print nothing. `fstrim` releases the space that
   deleted build trees used inside the image, so the image file shrinks
   inside WSL again.
4. Copy `$IMG` to the Windows side and write it with Rufus (DD mode) or
   balenaEtcher, as in
   [wsl2-build-host.md](wsl2-build-host.md#4-write-an-image-to-a-usb-stick).
   **Pick the stick, not another drive.** The copy on C: is a full 20 GiB.

**The image in WSL is the master copy.** Anything you change on the stick is
lost the next time you write the image. Make changes in the chroot and write
the stick again. Once you have OpenSSH, you can instead copy a single file,
such as a new kernel, across with `scp`, and make the same change in the
image.

---

## 8. Boot it on the NUC

Plug in the stick and power on. Press **F10** for the boot menu and pick the
stick's UEFI entry. **Secure Boot must be off**: LFS has no way to sign the
kernel (the book says so in 10.4.2). See
[BIOS settings](nuc10.md#bios-settings) and
[Booting from USB](nuc10.md#booting-from-usb) in nuc10.md.

You should see kernel messages, then systemd's `[  OK  ]` lines, then:

```
lfs login:
```

Log in as root. Type `æøå` at the prompt: if the Danish keymap is active,
the letters appear as typed.

The kernel may also print `GPT:Primary header thinks Alt. header is not at
the end of the disk.` This happens because the image is smaller than the
stick, and it's harmless.

If it goes wrong:

| What you see | What to check |
|---|---|
| The stick isn't in the F10 menu | Secure Boot is still on, or `EFI/BOOT/BOOTX64.EFI` is missing from the ESP. Some USB SSD enclosures report 4096-byte sectors, and the image's partition table (made for 512-byte sectors) isn't found on them; try a stick |
| `Waiting for root device PARTUUID=...` and nothing more | The UUID in `CONFIG_CMDLINE` doesn't match the image's root partition, or one of the USB options isn't built in (`=y`). Run the grep from section 6 again |
| Kernel panic, `VFS: Unable to mount root fs` | `EXT4_FS` isn't built in, or the command line is wrong |
| The screen goes black after the first kernel messages | The book's framebuffer options are missing: `DRM_FBDEV_EMULATION`, `FRAMEBUFFER_CONSOLE`, `DRM_SIMPLEDRM` |
| Emergency mode | Usually a mistake in `/etc/fstab`. The emergency shell lets you fix it |
| A long wait on "Wait for Network to be Configured" | No cable, or the `.network` file doesn't match the port |

The book's [troubleshooting page](https://www.linuxfromscratch.org/lfs/troubleshooting.html)
covers more.

---

## 9. First checks

All of these work on a bare LFS system. `lspci` and `lsusb` don't: they come
from pciutils and usbutils in BLFS.

```bash
uname -r                     # 7.1.8
findmnt /                    # /dev/sdX2, ext4, rw
lsblk                        # the stick (sdX) and the NUC's nvme0n1
networkctl                   # the en... link: routable, configured
ip -4 addr                   # an address from your router
ping -c 3 www.linuxfromscratch.org
localectl                    # VC Keymap: dk
timedatectl                  # Time zone: Europe/Copenhagen
systemctl --failed           # 0 loaded units listed
lsmod | grep -E 'i915|iwlmvm'
journalctl -k -b | grep -E 'DMC firmware|loaded firmware version'
```

**Leave `nvme0n1` alone.** It's the NUC's own disk.

The last command should show two lines. One is from i915:
`Finished loading DMC firmware i915/kbl_dmc_ver1_04.bin (v1.4)`. The other
is from iwlwifi and ends in `QuZ-a0-hr-b0-77.ucode op_mode iwlmvm`. If you
see `Direct firmware load for ... failed with error -2` instead, the file
named in the message is missing from `/usr/lib/firmware`. A `wl...`
interface in `ip link` means the Wi-Fi card is working, but you can't
connect without the BLFS tools.

---

## 10. Next steps

- **[BLFS 13.1-systemd](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/)**
  is the next book. Build its packages in the chroot in WSL, which is the
  book's own first suggestion in
  [11.5.2.1](https://www.linuxfromscratch.org/lfs/view/13.1-systemd/chapter11/afterlfs.html),
  and write the stick again. Download sources on the Ubuntu side, since LFS
  has no `wget`.
- **[OpenSSH](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/postlfs/openssh.html)**
  has no required dependencies, so it's a good first package. Its page shows
  how to install `sshd.service` from blfs-systemd-units.
- **Wi-Fi** needs
  [wpa_supplicant](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/basicnet/wpa_supplicant.html).
  BLFS 13.1 doesn't include iwd.
- **`lspci` and `lsusb`:**
  [pciutils](https://www.linuxfromscratch.org/blfs/view/13.1-systemd/general/pciutils.html)
  and usbutils.
- **Intel microcode (optional).** Run `lscpu` on the NUC to see the CPU's
  family, model and stepping. Intel's release note lists Comet Lake-U as
  `06-8e-0c` for the 4-core die (i3 and i5) and `06-a6-00` or `06-a6-01` for
  the 6-core i7. Get the file from
  [Intel's releases](https://github.com/intel/Intel-Linux-Processor-Microcode-Data-Files/releases/)
  (`microcode-20261110` on 2026-11-19), put it in
  `$LFS/usr/lib/firmware/intel-ucode/`, and build it into the kernel:
  `scripts/config --set-str EXTRA_FIRMWARE "intel-ucode/<XX-YY-ZZ>"`, then
  `make olddefconfig`, `make`, and copy `bzImage` to the ESP again. The
  kernel docs describe this as "builtin microcode". After the next boot,
  `journalctl -k -b | grep microcode` should show `Updated early from`.
  BLFS's About Firmware page has the details, and how to tell whether the
  BIOS has already applied the update.
- **Disk space:** when you're done, delete the image and the copy on C:, then
  compact WSL's disk
  ([wsl2-build-host.md](wsl2-build-host.md#5-get-the-disk-space-back)).
