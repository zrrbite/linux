# WSL2 as the build host

LFS and Yocto both need a Linux machine to build on, and the NUC's disk is
too small for either build. So both build on the Windows workstation in WSL2,
and the result goes to the NUC on a USB stick. This page is the part the two
guides share: the Ubuntu install they build in, writing an image to a stick,
and getting the disk space back afterwards. Zephyr builds natively on Windows
and doesn't need any of this.

Checked against **WSL 2.6.3** (what the workstation runs on 2026-11-19) and
the [WSL docs](https://learn.microsoft.com/en-us/windows/wsl/).

| | The workstation on 2026-11-19 |
|---|---|
| CPU | i7-12700K, 12 cores / 20 threads |
| RAM | 32 GB. WSL gets half, 16 GB, unless told otherwise |
| Free disk | C: 232 GB, F: 81 GB |
| Existing WSL distro | `Ubuntu`, which is **20.04**: too old for current LFS and Yocto. Leave it alone; the builds get their own Ubuntu 24.04 next to it |
| `/dev/kvm` in WSL | Present, so QEMU in WSL runs with hardware acceleration |

---

## 1. Install WSL2 and Ubuntu

In PowerShell:

```powershell
wsl --update
wsl --install -d Ubuntu-24.04
```

It opens the new distro and asks for a Linux username and password. Then
check, in PowerShell:

```powershell
wsl -l -v
```

You should see `Ubuntu-24.04` with VERSION `2`, next to the old `Ubuntu`.
The `*` marks the default, which is what a plain `wsl` opens. It's still the
old 20.04, so either open the new one by name (`wsl -d Ubuntu-24.04`, or pick
it in Windows Terminal's dropdown) or make it the default:

```powershell
wsl --set-default Ubuntu-24.04
```

Inside the new distro, update it:

```bash
sudo apt update && sudo apt full-upgrade -y
lsb_release -d
```

You should see `Ubuntu 24.04.x LTS`. Each guide lists the packages it needs
on top of this.

## 2. Memory, CPUs and disk

WSL reads its settings from `.wslconfig` in your Windows user folder
(`C:\Users\<you>\.wslconfig`). There isn't one yet. Create it with:

```ini
[wsl2]
memory=16GB
processors=20
swap=16GB
```

Why: 16 GB is WSL's default here (half the RAM), written down so you can see
it. The setting that matters is `swap`: by default WSL gets only a quarter
of its memory as swap, and a build that runs out of memory dies halfway.
With 16 GB of swap, a Yocto build that briefly needs more slows down
instead. Yocto's docs ask for 32 GB of RAM; the Yocto guide runs fewer jobs
at once to fit in 16. LFS needs far less. `processors=20` is the default
anyway; it's there so you know where to turn it down if Windows gets
sluggish during a build.

**Don't give WSL much more than half.** The first Yocto build here ran with
`memory=24GB`. WSL's VM grew to 21.6 GB, much of it Linux's file cache,
which it doesn't hand back to Windows while a build runs. Windows was left
with 0.9 GB free and the build had to be stopped at task 6,244 of 6,335. It
resumed fine with 16 GB.

The settings take effect after WSL restarts:

```powershell
wsl --shutdown
```

Then open the distro again. You should see `nproc` print `20`, and `free -g`
show about 23 GB total and 16 GB swap.

**Disk:** the distro's whole filesystem is one file, `ext4.vhdx`, on C:. It
grows as the builds write to it and doesn't shrink by itself when you delete
things ([section 5](#5-get-the-disk-space-back)). What the builds need:

| Build | Inside WSL | On C: besides |
|---|---|---|
| LFS | About 40 GB: the 20 GiB image, sources, a backup | 20 GB for the image copy you write to the stick |
| Yocto | Up to 140 GB, or about 40 GB with `rm_work` ([Yocto guide, step 5](yocto-guide.md#5-build)) | The `.wic` copy, a few GB |

C: has room for either build, and for both if Yocto runs with `rm_work`.
F: is too small for Yocto, so leave the distro where it is. If C: ever gets
tight, `wsl --manage Ubuntu-24.04 --move <folder>` moves the distro to
another drive (run `wsl --shutdown` first).

## 3. Where to keep build trees

Keep everything in the Linux filesystem, under `~` (`/home/<you>`), never
under `/mnt/c`. `/mnt/c` is the Windows drive reached through a translation
layer: builds there are many times slower, file names are case-insensitive,
and permissions and symlinks don't behave the way the build tools expect.

To get at the files from Windows, run `explorer.exe .` in a folder, or open
`\\wsl.localhost\Ubuntu-24.04\home\<you>` in Explorer.

## 4. Write an image to a USB stick

The LFS and Yocto guides each end with a disk image: a file holding a whole
disk, partition table and all. Copy it to the Windows side (`<you>` is your
Windows user name):

```bash
cp <image file> /mnt/c/Users/<you>/Downloads/
```

Yocto's images end in `.wic`. Rufus's file picker doesn't show `.wic` files,
so give the copy an `.img` name; the contents are the same raw disk image.

Then write it with [Rufus](https://rufus.ie/):

1. Plug in the stick. Note its size, so you can tell it from other drives.
2. Start Rufus. Under **Device**, pick the stick. **Check the name and size
   twice: everything on the drive you pick is erased.**
3. Under **Boot selection**, click **SELECT** and pick the `.img`. Rufus sees
   that it's a disk image and writes it as is (DD mode); the partition and
   file system options grey out.
4. Click **START** and confirm.

[balenaEtcher](https://etcher.balena.io/) does the same: Flash from file,
Select target, Flash. It also reads the stick back afterwards to check it.

Afterwards Windows may say the drive needs to be formatted. Click **Cancel**:
Windows can't read the Linux partitions, and formatting would erase them.

Then boot the NUC from it: [nuc10.md](nuc10.md#booting-from-usb).

## 5. Get the disk space back

`ext4.vhdx` stays at its biggest size after you delete files inside it. Two
ways to get the space back.

**Shrink it, keeping the distro.** Delete what you don't need inside WSL,
then tell the virtual disk which blocks are now free:

```bash
sudo fstrim -av
```

Find the folder that holds the distro's `ext4.vhdx`, in PowerShell:

```powershell
(Get-ChildItem HKCU:\Software\Microsoft\Windows\CurrentVersion\Lxss |
  Get-ItemProperty | Where-Object DistributionName -eq 'Ubuntu-24.04').BasePath
```

Then, in a PowerShell window run as administrator:

```powershell
wsl --shutdown
diskpart
```

and at the `DISKPART>` prompt, with the folder from above:

```
select vdisk file="<folder>\ext4.vhdx"
attach vdisk readonly
compact vdisk
detach vdisk
exit
```

`Optimize-VHD` does the same, but needs the Hyper-V PowerShell module;
`diskpart` is always there. WSL's "sparse" disks would shrink by themselves,
but after reports of data corruption WSL 2.6 won't turn them on without an
`--allow-unsafe` flag. Don't use them for a disk that holds days of build
work.

**Done with the builds for good:** this deletes the distro and its
`ext4.vhdx`, and with them everything inside. **Copy out anything you want to
keep first**: images, your own layers and configs.

```powershell
wsl --unregister Ubuntu-24.04
```
