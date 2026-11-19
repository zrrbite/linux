# Zephyr on the NUC 10

[Zephyr](https://www.zephyrproject.org/) is a real-time operating system,
mainly for microcontrollers, that also runs on 64-bit x86. It isn't Linux:
you build one application image, and there's no shell, filesystem or
processes unless you build them in. This guide builds Zephyr samples on the
Windows workstation (or the Mac), runs them in QEMU, then boots them on the
NUC 10 from a USB stick with the output on the HDMI screen.

Checked against **Zephyr 4.5.0** (released 2026-10-28), **Zephyr SDK 1.0.1**
(released 2026-03-25), the 4.5.0
[Getting Started Guide](https://docs.zephyrproject.org/4.5.0/develop/getting_started/index.html)
and the [v4.5.0 source](https://github.com/zephyrproject-rtos/zephyr/tree/v4.5.0).
Nothing here needs Zephyr's `main` branch.

**Status:** written 2026-11-19. Steps 1 to 5 and the step 8 build were run
on the Windows workstation that day, including QEMU; not yet tried on the
NUC.

Nothing is built on the NUC, and nothing is written to its NVMe drive: the
image goes on a USB stick, and the samples have no storage drivers.

---

## How it boots, and how sure this is

**There's no NUC 10 board in Zephyr**, and no Comet Lake one. The 64-bit x86
boards in 4.5.0 are mostly Intel's reference boards (Alder Lake N, Raptor
Lake, Elkhart Lake, Bartlett Lake, Panther Lake, Wildcat Lake, Nova Lake),
two UP boards, and QEMU's `qemu_x86_64`. Almost all the real-hardware ones
boot the same way: Zephyr is built as a UEFI application, `zephyr.efi`
(`CONFIG_BUILD_OUTPUT_EFI`), which the firmware starts like a boot loader.
A small loader inside it copies Zephyr into low memory and jumps to it.

This guide borrows the **Alder Lake N reference board, `intel_adl_crb`**
([source](https://github.com/zephyrproject-rtos/zephyr/tree/v4.5.0/boards/intel/adl)),
and changes three settings on the build command line. It's picked over the
other Intel boards because it turns ACPI on, so Zephyr finds the CPUs in the
NUC's own ACPI tables instead of in a fixed list written for another chip.

**Output:** the NUC 10 has no serial port and no UART header. Intel's
[technical product specification](https://dlcdnets.asus.com/pub/ASUS/NUC/TPS/NUC10i357FN_TechProdSpec.pdf)
(Rev 003, April 2021) lists fan, front panel, RGB LED, microphone, CEC and two
USB 2.0 headers, and nothing else. So the build switches Zephyr's console to
the **UEFI firmware's own text console**, which is what you see on the HDMI
screen in the boot menu. This works because `zephyr.efi` never calls
`ExitBootServices()`: the firmware stays in memory, and Zephyr's EFI console
(`CONFIG_X86_EFI_CONSOLE`, in `arch/x86/core/efi.c`) hands each line to the
firmware's `ConOut->OutputString()`. It's text only and output only, and
anything printed from an interrupt handler is dropped.

Not used: Zephyr's multiboot framebuffer driver, for booting through GRUB.
It's a display driver, not a text console, the Alder Lake board's devicetree
doesn't include it, and it only accepts a framebuffer below 4 GB.

**How sure:** the Windows builds and the QEMU runs were checked on the
workstation; QEMU on native Windows needs a `PATH` fix
([step 4](#4-run-it-in-qemu-first)). The NUC part is an experiment: each piece is in Zephyr 4.5.0, and the EFI
boot path is the one Intel's boards use, but no board turns the EFI console
on by default, and there's no report of this combination (another board's
target, the EFI console, Comet Lake firmware). It may need one of the fixes in
[If it doesn't work](#if-it-doesnt-work).

---

## Before you start

- **Windows 11**, or an **Apple Silicon Mac**. Zephyr no longer supports
  x86-64 macOS.
- **Disk space** on the build machine for Zephyr, its modules and the SDK.
  `west update` fetches dozens of module repositories.
- **A USB stick**, any size. Step 6 erases it.
- **An HDMI screen and a USB keyboard** on the NUC. The keyboard is only for
  F2 and F10.
- **Secure Boot off** on the NUC: `zephyr.efi` isn't signed. See
  [nuc10.md](nuc10.md#bios-settings).
- **Which CPU the NUC has**, for step 5. The model number (on the box and the
  label underneath) says it: NUC10i3FNK, NUC10i5FNK or NUC10i7FNK.

---

## 1. Install the build tools (Windows)

Open Windows Terminal (PowerShell) as your normal user, not as
administrator. The Getting Started Guide also has `cmd.exe` versions of every
command.

```powershell
winget install Kitware.CMake Ninja-build.Ninja oss-winget.gperf Python.Python.3.12 Git.Git oss-winget.dtc wget 7zip.7zip
```

Then close the terminal and open a new one, so it picks up the new `PATH`.
Use Python 3.12 even if a newer one is installed: the Getting Started Guide
warns that newer releases may fail to install Zephyr's packages on Windows.

Leave out what you already have. On the workstation, CMake, Git and 7-Zip
came from [scoop](https://scoop.sh/), and that works as well. On 2026-11-19
winget's `wget` package failed its installer hash check; nothing in this
guide needed it.

## 2. Get Zephyr 4.5.0

```powershell
cd $Env:HOMEPATH
py -3.12 -m venv zephyrproject\.venv
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
zephyrproject\.venv\Scripts\Activate.ps1
pip install west
west init -m https://github.com/zephyrproject-rtos/zephyr zephyrproject --mr v4.5.0
cd zephyrproject
west update
python -m pip install @((west packages pip) -split ' ')
west zephyr-export
```

`Set-ExecutionPolicy` is needed once, so PowerShell may run the venv's
activation script. Skip it if `Get-ExecutionPolicy -List` already shows
`RemoteSigned` for `LocalMachine`, as it does on the workstation. `--mr v4.5.0` pins the release tag instead of `main`.
`west update` takes a while.

You should see the prompt start with `(.venv)`, and
`git -C zephyr describe --tags` print `v4.5.0`.

In every new terminal, activate the venv again before using `west`:

```powershell
cd $Env:HOMEPATH\zephyrproject
.venv\Scripts\Activate.ps1
```

## 3. Install the Zephyr SDK

The SDK holds the compilers. Install only the x86-64 one: a 56 MiB download
instead of the 1.9 GiB bundle with every architecture. The SDK's setup script
needs `7z` on the `PATH`, which the 7-Zip install from winget may not have
done:

```powershell
$Env:PATH += ";C:\Program Files\7-Zip"
cd $Env:HOMEPATH\zephyrproject\zephyr
west sdk install -t x86_64-zephyr-elf
west sdk list
```

`west sdk install` reads the version (1.0.1) from Zephyr's `SDK_VERSION` file
and installs to `C:\Users\<you>\zephyr-sdk-1.0.1`. You should see:

```
1.0.1:
  path: C:\Users\<you>\zephyr-sdk-1.0.1
  hosttools: installed
  gnu-installed-toolchains:
    - x86_64-zephyr-elf
```

followed by a list of the toolchains you didn't install. The install itself
prints `SKIPPED: Windows host tools are not available yet.` Ignore it: the
host tools are installed anyway, and QEMU is in
`zephyr-sdk-1.0.1\hosttools\qemu`.

### On the Mac

The same steps, with [Homebrew](https://brew.sh/) instead of winget and Unix
paths:

```bash
brew install cmake ninja gperf python3 python-tk ccache qemu dtc libmagic wget openocd
(echo; echo 'export PATH="'$(brew --prefix)'/opt/python/libexec/bin:$PATH"') >> ~/.zprofile
source ~/.zprofile

python3 -m venv ~/zephyrproject/.venv
source ~/zephyrproject/.venv/bin/activate
pip install west
west init -m https://github.com/zephyrproject-rtos/zephyr ~/zephyrproject --mr v4.5.0
cd ~/zephyrproject
west update
west packages pip --install
west zephyr-export

cd ~/zephyrproject/zephyr
west sdk install -t x86_64-zephyr-elf
west sdk list
```

The SDK goes to `~/zephyr-sdk-1.0.1`. In later steps, use `/` instead of `\`
in paths, and `source ~/zephyrproject/.venv/bin/activate` in each new
terminal.

---

## 4. Run it in QEMU first

`qemu_x86_64` is QEMU's emulated q35 PC with two CPUs and a serial console.
It isn't the NUC, but it shows the tools and the samples work before any
hardware is involved.

```powershell
cd $Env:HOMEPATH\zephyrproject\zephyr
west build -p always -b qemu_x86_64 samples\hello_world
west build -t run
```

`-p always` starts each build from scratch, which matters when you switch
boards or options. You should see, possibly after a `Booting from ROM..`
line:

```
*** Booting Zephyr OS build v4.5.0 ***
Hello World! qemu_x86_64/atom
```

Press **Ctrl+A**, then **X**, to quit QEMU.

Then the sample used on the NUC in step 8. It works out 240 digits of pi in
16 threads at once and times them:

```powershell
west build -p always -b qemu_x86_64 samples\arch\smp\pi
west build -t run
```

You should see 16 lines starting `Pi value calculated by thread #`, then
`All 16 threads executed by 2 cores in ... msec`.

**On the Mac**, Zephyr's instructions install QEMU (from Homebrew, and again
in the SDK), so this should work as shown.

### Native Windows: add two folders from Git first

Since 1.0.0 the SDK ships QEMU for Windows too (QEMU 10.0.2, in
`zephyr-sdk-1.0.1\hosttools\qemu`), and the build finds it there. But
`west build -t run` fails in a plain PowerShell, in two ways:

- The run target pipes an `objcopy` step through `grep` and `true`, which
  Windows doesn't have: `'grep' is not recognized as an internal or external
  command`.
- QEMU itself needs `libgcc_s_seh-1.dll` and `libwinpthread-1.dll`, which
  the SDK leaves out. Without them it exits at once with code `3221225781`
  (`0xC0000135`, a DLL not found), as Zephyr issue
  [#107534](https://github.com/zephyrproject-rtos/zephyr/issues/107534)
  reports. The issue was closed as stale on 2026-08-09 without a fix.

Git for Windows has both DLLs in `mingw64\bin`, and `grep` and `true` in
`usr\bin`. Add both folders to this terminal's `PATH`; one isn't enough:

```powershell
$Env:PATH += ";C:\Program Files\Git\mingw64\bin;C:\Program Files\Git\usr\bin"
west build -p always -b qemu_x86_64 samples\hello_world
west build -t run
```

With Git from scoop, the folders are
`$Env:USERPROFILE\scoop\apps\git\current\mingw64\bin` and `...\usr\bin`.
Checked on 2026-11-19: with both folders, hello_world and the pi sample ran
in QEMU from PowerShell with the output shown above (pi: `All 16 threads
executed by 2 cores in 15 msec`). If it still fails for you, run QEMU on the
Mac or in WSL2.

### In WSL2

Use the Ubuntu 24.04 set up in [wsl2-build-host.md](wsl2-build-host.md) and
Zephyr's Ubuntu instructions. Keep the workspace in the Linux home directory,
not under `/mnt/c`, where builds are much slower.

```bash
sudo apt update
sudo apt install --no-install-recommends git cmake ninja-build gperf \
  ccache dfu-util device-tree-compiler wget python3-dev python3-venv python3-tk \
  xz-utils file make gcc gcc-multilib g++-multilib libsdl2-dev libmagic1

python3 -m venv ~/zephyrproject/.venv
source ~/zephyrproject/.venv/bin/activate
pip install west
west init -m https://github.com/zephyrproject-rtos/zephyr ~/zephyrproject --mr v4.5.0
cd ~/zephyrproject
west update
west packages pip --install
west zephyr-export

cd ~/zephyrproject/zephyr
west sdk install -t x86_64-zephyr-elf
west build -p always -b qemu_x86_64 samples/hello_world
west build -t run
```

On Linux the SDK installs its own QEMU, so there's nothing else to install.
The NUC builds in steps 5 to 8 still happen on Windows.

---

## 5. Build hello_world for the NUC

Look up your CPU's base clock. Zephyr's timer counts the CPU's time stamp
counter (TSC), which runs at about the base clock, and Zephyr doesn't measure
it: it takes the number from the build. The Alder Lake N board assumes
1.5 GHz. The figures are from Intel's specification:

| NUC | CPU | Base clock | Value |
|---|---|---|---|
| NUC10i3FNK | Core i3-10110U, 2 cores, 4 threads | 2.1 GHz | `2100000000` |
| NUC10i5FNK | Core i5-10210U, 4 cores, 8 threads | 1.6 GHz | `1600000000` |
| NUC10i7FNK | Core i7-10710U, 6 cores, 12 threads | 1.1 GHz | `1100000000` |

The commands below use the i5's value; change it if yours differs. A wrong
value only makes sleeps and timings wrong.

```powershell
cd $Env:HOMEPATH\zephyrproject\zephyr
west build -p always -b intel_adl_crb samples\hello_world -- -DCONFIG_UART_CONSOLE=n -DCONFIG_SERIAL=n -DCONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=1600000000
```

Everything after `--` goes to CMake, and `-DCONFIG_...` sets a Kconfig option
for this build:

- `CONFIG_UART_CONSOLE=n` turns off the serial console. With it off,
  `CONFIG_X86_EFI_CONSOLE` defaults to on, and Zephyr prints through the
  firmware's console.
- `CONFIG_SERIAL=n` turns off the UART driver. The board's devicetree points
  at an Alder Lake UART found by its PCI ID (8086:54a8) and a legacy port at
  I/O 0x3f8; the NUC has neither, so don't probe them.
- `CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC` is the TSC rate from the table.

You should see the build finish, a file `build\zephyr\zephyr.efi` of about
2.4 MB, and `CONFIG_X86_EFI_CONSOLE=y` among the lines this prints. The
build's last lines include `objcopy: ... section .trash_bin lma ... adjusted
to ...`, which is harmless. The RAM line shows about 69 MB used: that's the
64 MB ACPI heap ([below](#whats-borrowed-from-alder-lake-n-and-what-could-break)).

```powershell
Select-String EFI_CONSOLE build\zephyr\.config
```

On the Mac: `grep EFI_CONSOLE build/zephyr/.config`.

## 6. Write the USB stick

The firmware starts `\EFI\BOOT\BOOTX64.EFI` from a FAT32 stick by itself,
without a boot loader, so that's where `zephyr.efi` goes. Windows only formats
FAT32 up to 32 GB, so the stick gets a 1 GB partition, which works with a
stick of any size.

**This erases the stick. Check the disk number twice:** `clean` wipes the
selected disk at once, without asking, and disk 0 is usually the
workstation's own system disk.

```powershell
diskpart
```

It asks for administrator rights and opens its own window. There:

```
list disk
select disk 2
detail disk
clean
create partition primary size=1024
format fs=fat32 quick label=ZEPHYR
assign
list volume
exit
```

`list disk` shows each disk with its size; pick the stick's number instead of
`2`. `detail disk` shows the selected disk's name and `Type : USB`: if it
doesn't, stop and select again. `list volume` shows the letter `ZEPHYR` got.
Back in PowerShell, with that letter instead of `E:`:

```powershell
cd $Env:HOMEPATH\zephyrproject\zephyr
mkdir E:\EFI\BOOT
copy build\zephyr\zephyr.efi E:\EFI\BOOT\BOOTX64.EFI
copy build\zephyr\zephyr.efi E:\zephyr.efi
```

The second copy is only for starting it by hand from a UEFI shell (see
[step 7](#7-boot-the-nuc)). Eject the stick from File Explorer.

**On the Mac:**

```bash
diskutil list                      # find the stick: "(external, physical)", e.g. /dev/disk4
diskutil eraseDisk FAT32 ZEPHYR MBRFormat /dev/disk4
mkdir -p /Volumes/ZEPHYR/EFI/BOOT
cp ~/zephyrproject/zephyr/build/zephyr/zephyr.efi /Volumes/ZEPHYR/EFI/BOOT/BOOTX64.EFI
cp ~/zephyrproject/zephyr/build/zephyr/zephyr.efi /Volumes/ZEPHYR/zephyr.efi
diskutil eject /dev/disk4
```

**Check the disk number twice** here too: `eraseDisk` erases whichever disk
you name.

## 7. Boot the NUC

Secure Boot must be off (F2, see [nuc10.md](nuc10.md#bios-settings)). Plug
in the stick, power on, press **F10** for the boot menu and pick the USB
stick ([nuc10.md](nuc10.md#booting-from-usb)).

You should see the EFI loader's lines, then Zephyr's:

```
*** Zephyr EFI Loader ***
RSDP found at 0x...
Zeroing ... bytes of memory at 0x...
Copying ... data bytes to 0x... from image offset ...
...
Jumping to Entry Point: 0x... (...)
*** Booting Zephyr OS build v4.5.0 ***
Hello World! intel_adl_crb/alder_lake
```

The loader's lines show the stick boots; the last two show Zephyr is running
and its console works. After that Zephyr idles. It has no keyboard or power
button handling, so **hold the power button for four seconds** to switch the
NUC off.

**UEFI shell, not confirmed:** Intel's BIOS-update instructions for earlier
NUCs mention an *Internal UEFI Shell* option (Advanced > Boot > Boot
Configuration) and a *UEFI: Built-in EFI Shell* entry in the F10 menu. The
NUC 10's specification doesn't mention one. You don't need it, but if your
F10 menu has it, you can start the copy in the stick's root by hand:
`map -r` lists the drives, then for example `fs0:` and `zephyr.efi`.

## 8. Pi on all the cores

The pi sample from step 4, on four of the NUC's logical CPUs:

```powershell
cd $Env:HOMEPATH\zephyrproject\zephyr
west build -p always -b intel_adl_crb samples\arch\smp\pi -- -DCONFIG_UART_CONSOLE=n -DCONFIG_SERIAL=n -DCONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC=1600000000 -DCONFIG_MP_MAX_NUM_CPUS=4
copy build\zephyr\zephyr.efi E:\EFI\BOOT\BOOTX64.EFI
```

On the Mac, copy it with `cp` as in step 6. Don't go above 4 on this target:
the Alder Lake SoC code has room for four APIC IDs. Zephyr takes the four
from the NUC's ACPI tables, so they may be four cores or two cores with two
threads each.

Boot it as in step 7. You should see the 16 `Pi value calculated by thread #`
lines, then `All 16 threads executed by 4 cores in ... msec`. For
comparison, build again with `-DCONFIG_MP_MAX_NUM_CPUS=1`: the same work on
one CPU should take several times as long.

---

## If it doesn't work

| What you see | Likely cause | What to try |
|---|---|---|
| The stick isn't in the F10 menu | USB boot settings, or the stick's layout | [nuc10.md](nuc10.md#booting-from-usb); check the file is `EFI\BOOT\BOOTX64.EFI` on a FAT32 partition |
| A Secure Boot message, or straight back to the menu | Secure Boot is on | Turn it off (F2) |
| Nothing after `Jumping to Entry Point` | Zephyr stopped before its first line, for example because its memory overlaps something the firmware still uses (see below) | Add `-DCONFIG_ACPI=n` to the build command: the image gets much smaller |
| Stops after `*** Booting Zephyr OS build v4.5.0 ***` | The second CPU didn't start: that happens between the banner and `main()` | Add `-DCONFIG_MP_MAX_NUM_CPUS=1`. If that works, try 2 CPUs with `-DCONFIG_X2APIC=n` |
| The pi sample stops part-way | Zephyr called the firmware's console from a CPU other than the first | Check with `-DCONFIG_MP_MAX_NUM_CPUS=1` |
| The `msec` figure is far off | Wrong TSC rate | Check `CONFIG_SYS_CLOCK_HW_CYCLES_PER_SEC` against the table in step 5 |

If no Zephyr line ever appears, there's no other confirmed way to get output
from Zephyr on this NUC; the QEMU runs in step 4 are the fallback.

### What's borrowed from Alder Lake N, and what could break

- **Memory.** The loader copies Zephyr to fixed addresses in low memory,
  starting at 4 KiB, without asking the firmware for them. Intel's notes
  (`arch/x86/zefi/README.txt`) say there's no check for collisions; it works
  on the machines they tried because firmware keeps its own data near 4 GB.
  With ACPI on, this target also adds a 64 MB heap to that image, which is
  why `CONFIG_ACPI=n` is the first fix to try.
- **ACPI.** The loader passes the firmware's ACPI table address to Zephyr,
  which reads the CPU list (the MADT) with ACPICA and starts only CPUs the
  table marks as enabled. With `CONFIG_ACPI=n`, Zephyr instead uses the
  Alder Lake SoC's fixed APIC IDs (0, 2, 4, 6); APIC ID 2 should exist on all
  three NUC 10 CPUs, so 2 CPUs should still start.
- **APIC and x2APIC.** The target uses x2APIC mode, which Comet Lake
  supports, and starts the other CPUs with the usual INIT and STARTUP
  interrupts.
- **Timer.** The local APIC in TSC-deadline mode, at the rate set in step 5.
  The HPET isn't used; the loader switches off its timer 0 interrupt before
  jumping to Zephyr.
- **UART.** The devicetree describes Alder Lake's serial ports, which the
  NUC doesn't have. That's why `CONFIG_SERIAL=n`.
- **Instructions.** The target compiles with `-march=goldmont`, an Atom
  instruction set that Comet Lake covers apart from the SHA instructions,
  which the compiler doesn't use on its own.
- **The firmware console from several CPUs.** UEFI firmware is meant to be
  called from the boot CPU only, and Zephyr's EFI console calls it from
  whichever CPU prints. hello_world most likely prints from the first CPU;
  the pi sample's main thread may move to another.

---

## Not confirmed

- That any of this boots on the NUC 10. The builds and the QEMU runs were
  checked on the workstation (status above); the NUC part wasn't.
- Whether the NUC 10's firmware has a built-in UEFI shell.
- Whether calling the firmware console from a second CPU works on this
  firmware.
- The TSC rate matching the base clock exactly; it's close enough for the
  timings here.

## Next: a board of its own

If hello_world works, the clean next step is an out-of-tree board: a
`boards/` folder in your own repo, passed with `-DBOARD_ROOT=<folder>`,
copied from `boards/intel/adl` with the EFI console, the right TSC rate and no
serial ports, so the `-D` options above go away. It isn't written here,
because none of it has been tried yet.
