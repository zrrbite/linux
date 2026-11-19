# linux

Guides for putting operating systems built from source on an Intel NUC10FNK:
Linux From Scratch, a Yocto image, and Zephyr, which is an RTOS rather than
Linux. The sibling [archinstall](https://github.com/zrrbite/archinstall) repo
covers Arch and Omarchy on the NUCs.

Nothing is built on the NUC, whose disk is small. LFS and Yocto build in WSL2
on the Windows workstation, and Zephyr builds natively on Windows or a Mac.
Each result boots on the NUC from a USB stick, so the NUC's own SSD is never
written.

All three guides were written on 2026-11-19 and haven't been tried on the
NUC yet. Zephyr is the quickest to try, and LFS takes the longest.

| Guide | For |
|---|---|
| [NUC10FNK](doc/nuc10.md) | The hardware, the BIOS settings, and booting from USB |
| [WSL2 build host](doc/wsl2-build-host.md) | The Ubuntu 24.04 that LFS and Yocto build in, writing an image to a stick, and getting the disk space back |
| [Linux From Scratch](doc/lfs-guide.md) | LFS 13.1-systemd, built into a disk image in WSL2 and started by the kernel's own EFI stub. Several evenings |
| [Yocto](doc/yocto-guide.md) | A Yocto 6.0 "Wrynose" (LTS) console image with SSH, for `genericx86-64`. One long first build, then quick rebuilds |
| [Zephyr](doc/zephyr-guide.md) | Zephyr 4.5 samples started as a UEFI program, with output on HDMI through the firmware's console. An experiment: Zephyr has no board for this NUC |
