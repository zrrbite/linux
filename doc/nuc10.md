# Intel NUC10FNK

The NUC the guides in this repo boot on: its hardware, the BIOS settings
they need, and how to boot it from a USB stick. Every guide here boots from
USB and leaves the NUC's own SSD alone, so pulling the stick puts the NUC
back to whatever it ran before.

Facts from Intel's
[technical product specification](https://dlcdnets.asus.com/pub/ASUS/NUC/TPS/NUC10i357FN_TechProdSpec.pdf)
(Rev 003, April 2021), now hosted by ASUS, which took over the NUC line.

## Specs

| | NUC10FNK |
|---|---|
| Model | NUC 10 Performance kit, slim chassis ("Frost Canyon"). The full model number, by CPU, is NUC10i3FNK, NUC10i5FNK or NUC10i7FNK: it's on the box and the label underneath. The taller NUC10FNH has a 2.5" drive bay |
| CPU | Comet Lake-U, 10th generation, soldered: **i3-10110U** (2 cores, 4 threads, 2.1 GHz base), **i5-10210U** (4 cores, 8 threads, 1.6 GHz), or **i7-10710U** (6 cores, 12 threads, 1.1 GHz) |
| Graphics | Intel UHD Graphics |
| Memory | Two DDR4-2666 SO-DIMM slots, up to 64 GB |
| Storage | One M.2 slot, key M, 2242 or 2280, NVMe (PCIe x4) or SATA. The board has a SATA port for a 2.5" drive, but the slim chassis has no room for one |
| Ethernet | Intel I219-V, 1 Gbit/s |
| Wi-Fi, Bluetooth | Intel Wi-Fi 6 AX201 with Bluetooth 5, soldered to the board |
| USB | All USB 3.1 Gen 2 (10 Gbit/s). Front: one Type-A, one Type-C. Back: two Type-A, and one Type-C that is also Thunderbolt 3 and DisplayPort |
| Display | HDMI 2.0b on the back, and DisplayPort through the back Type-C port |
| Serial port | None, and no UART header inside. The internal headers are fan, front panel, RGB LED, microphone, CEC and two USB 2.0 |
| Other | SDXC card slot, infrared receiver, firmware TPM (Intel PTT) |

To see which CPU yours has without opening the box: the BIOS (F2) shows it
on its main page, and on Linux `grep -m1 'model name' /proc/cpuinfo` does.

---

## BIOS settings

Press **F2** at power-on for the BIOS setup (Intel Visual BIOS).

- **Secure Boot off.** None of the guides signs what it boots: the LFS
  kernel, the Yocto image and `zephyr.efi` all stop at a Secure Boot check.
  It's among the boot settings; in Visual BIOS that's **Advanced > Boot >
  Secure Boot**. Not checked on this NUC's BIOS version, so look around if
  it's not there.
- **Leave the boot mode on UEFI.** All three guides boot through UEFI.
- **Leave the boot order alone.** By default the NUC tries the internal
  drive before removable ones, so it keeps starting its own system, and you
  pick the stick with F10 when you want it.

**If F2 or F10 don't seem to work:** with Fast Boot on, the NUC may skip the
keyboard check and go straight past them. Use the power button menu instead.
With the NUC off, hold the power button for **3 seconds**, until its light
turns from blue to amber, and let go before 4 seconds (4 seconds forces it
off). A menu appears:

| Key | Action |
|---|---|
| `Esc` | Normal boot |
| `F2` | BIOS setup |
| `F3` | Turn off Fast Boot (only shown when it's on) |
| `F4` | BIOS recovery |
| `F7` | Update BIOS |
| `F10` | Boot menu |
| `F12` | Network boot |

The menu doesn't work right after the power cable was unplugged. Power the
NUC on and off once first.

---

## Booting from USB

1. Plug the stick into a blue USB port, or the USB-C port.
2. Power on and press **F10** a few times, until the boot menu appears.
3. Pick the stick with the arrow keys and press **Enter**. It's listed by its
   make, as a UEFI entry. **Esc** leaves the menu and boots normally.

To go back to the NUC's own system, power off, pull the stick, and power on.
Holding the power button for 4 seconds switches the NUC off from anything,
including Zephyr, which has no shutdown of its own.

If the stick isn't in the menu:

- Secure Boot is still on.
- The stick has no `EFI\BOOT\BOOTX64.EFI` on a FAT partition. That's the
  file UEFI looks for on a removable drive, and every guide here puts one
  there. Write the stick again.
- Some USB SSD enclosures report 4096-byte sectors, and an image made for
  512-byte sectors isn't found on them. Try a plain stick.

| Guide | What's on the stick | What the NUC starts |
|---|---|---|
| [LFS](lfs-guide.md) | A 20 GiB image: an EFI partition and an ext4 root | The Linux kernel itself, as `BOOTX64.EFI` |
| [Yocto](yocto-guide.md) | The `.wic` image Yocto builds: an EFI partition, an ext4 root and a small swap partition | GRUB, as `BOOTX64.EFI`, which starts the kernel |
| [Zephyr](zephyr-guide.md) | One small FAT32 partition | `zephyr.efi`, as `BOOTX64.EFI` |
