# Ubuntu VirtualBox VM → Physical Laptop (V2P Migration)

A step-by-step guide to moving a working **Ubuntu** installation from a **VirtualBox VM** onto a **physical laptop**, using **Rescuezilla** for the disk image. This process is called a **Virtual-to-Physical (V2P)** migration.

> **Result:** The same OS, packages, users and settings from the VM, running on real hardware. No fresh install needed.

---

## Table of Contents

1. [How it works](#how-it-works)
2. [Requirements](#requirements)
3. [Step 1 – Audit the VM](#step-1--audit-the-vm)
4. [Step 2 – Prepare the VM to boot on real hardware](#step-2--prepare-the-vm-to-boot-on-real-hardware)
5. [Step 3 – Snapshot and USB preparation](#step-3--snapshot-and-usb-preparation)
6. [Step 4 – Capture the VM image with Rescuezilla](#step-4--capture-the-vm-image-with-rescuezilla)
7. [Step 5 – Restore the image to the laptop](#step-5--restore-the-image-to-the-laptop)
8. [Post-migration checklist](#post-migration-checklist)
9. [Troubleshooting](#troubleshooting)
10. [Important notes](#important-notes)

---

## How it works

A VM disk is built for virtual hardware. A laptop has different hardware, and often a different boot method (UEFI instead of BIOS). So the plan is:

```
[Ubuntu VM]
    |  1. Audit boot mode, partitions, fstab, network
    |  2. Add real-hardware drivers + a bootloader that works on BIOS and UEFI
    |  3. Take a snapshot (safety net) + prepare two USB drives
    |  4. Boot Rescuezilla inside the VM and save the disk as an image
    v
[Image on 64 GB USB]
    |  5. Boot the laptop from Rescuezilla and restore the image
    v
[Ubuntu on physical laptop]
```

---

## Requirements

| Item | Details |
|---|---|
| Source | Ubuntu VM in VirtualBox (with sudo access) |
| Target | A laptop whose internal disk can be **fully erased** |
| USB 1 | 8 GB or more, used as the **Rescuezilla boot drive** (flash with Rufus or balenaEtcher) |
| USB 2 | 64 GB, used as **image storage** (format **exFAT** or **NTFS**) |
| VirtualBox | USB 3.0 (xHCI) support, which needs the **VirtualBox Extension Pack** |
| Network | Internet access inside the VM (to install packages) |

> **Why exFAT/NTFS for the storage USB?** FAT32 cannot store files larger than 4 GB. Disk images are usually bigger than that.

---

## Step 1 – Audit the VM

**Goal:** Know exactly how the VM boots and how its disk is laid out, before changing anything.

```bash
# 1. Boot mode
[ -d /sys/firmware/efi ] && echo "UEFI" || echo "Legacy BIOS"

# 2. Disks, partitions, filesystems, UUIDs
lsblk -o NAME,SIZE,FSTYPE,MOUNTPOINT,UUID

# 3. What mounts at boot (comments removed)
grep -v '^#' /etc/fstab

# 4. Partition table type and layout
sudo parted -l
```

### What each check tells you

| Command | What to look for | Why it matters |
|---|---|---|
| `/sys/firmware/efi` check | `UEFI` or `Legacy BIOS` | Decides which bootloader parts are missing. Many laptops only boot UEFI. |
| `lsblk` | Partition names, sizes, filesystem types, UUIDs | Gives you the layout to protect and the UUIDs used in `fstab`. |
| `/etc/fstab` | Entries using `UUID=...` | Partition imaging keeps filesystem UUIDs, so UUID-based entries keep working after restore. Remove or mark `nofail` any entry that will not exist on the laptop (for example VirtualBox shared folders). |
| `parted -l` | `Partition Table: msdos` (MBR) or `gpt`, plus free space | Shows where there is free space to add an EFI partition in Step 2. |
| Network config (`/etc/netplan/`, `ip a`) | Interface name such as `enp0s3` | VM network card names will not match the laptop. See [post-migration](#post-migration-checklist). |

---

## Step 2 – Prepare the VM to boot on real hardware

**Goal:** Make the VM's disk bootable on **both Legacy BIOS and UEFI** machines, and make sure real-hardware drivers are present.

### 2.1 Install bootloader packages and firmware

```bash
sudo apt update && sudo apt install -y \
  grub-efi-amd64-bin grub-efi-amd64-signed shim-signed \
  efibootmgr linux-firmware dosfstools
```

| Package | Purpose |
|---|---|
| `grub-efi-amd64-bin` | GRUB modules for 64-bit UEFI systems |
| `grub-efi-amd64-signed` | Signed GRUB, needed when Secure Boot is on |
| `shim-signed` | First-stage signed loader for Secure Boot |
| `efibootmgr` | Tool to view and edit UEFI boot entries |
| `linux-firmware` | Firmware files for real hardware (Wi-Fi, GPU, Bluetooth and more) |
| `dosfstools` | Provides `mkfs.vfat` to format the EFI partition |

### 2.2 Rebuild the initramfs for all kernels

```bash
sudo update-initramfs -u -k all
```

The initramfs is the small image the kernel loads first to find the root disk. Rebuilding it for **all** installed kernels makes sure the storage and hardware drivers needed at early boot are included, not only the ones the VM used.

### 2.3 Create an EFI System Partition (ESP)

A UEFI laptop needs a small FAT32 partition holding the bootloader. A BIOS-style VM often has none, so create one in free disk space:

```bash
sudo parted -s /dev/sda mkpart EFI fat32 43.0GB 43.6GB   # ~600 MB partition
sudo parted -s /dev/sda set 6 esp on                      # mark it as the ESP
sudo mkfs.vfat -F 32 -n EFI /dev/sda6                     # format as FAT32
sudo mkdir -p /boot/efi && sudo mount /dev/sda6 /boot/efi
```

> **Adjust these values for your disk.** `/dev/sda`, the start/end sizes (`43.0GB`, `43.6GB`) and the partition number (`6`) came from this VM's layout. Use the output of Step 1 (`sudo parted /dev/sda unit GB print free`) to pick yours. The number after `set` must match the partition you just created.

### 2.4 Install GRUB for UEFI (two ways)

```bash
# A) Removable/fallback path: EFI/BOOT/BOOTX64.EFI
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --no-nvram --removable

# B) Named Ubuntu entry: EFI/ubuntu/
sudo grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=ubuntu --no-nvram

sudo update-grub
```

| Option | Meaning |
|---|---|
| `--target=x86_64-efi` | Install the 64-bit UEFI version of GRUB |
| `--efi-directory=/boot/efi` | Put files on the new ESP |
| `--removable` | Writes to the default fallback path that **any UEFI firmware checks automatically**. This is the key to booting on a machine that has no saved boot entry for your OS. |
| `--bootloader-id=ubuntu` | Creates the normal `ubuntu` folder on the ESP |
| `--no-nvram` | Do not write boot entries into the VM's firmware. The laptop has its own firmware, so the VM's entries are useless. |
| `update-grub` | Regenerates the GRUB menu/config |

### 2.5 Mount the ESP automatically

```bash
echo "UUID=$(sudo blkid -s UUID -o value /dev/sda6) /boot/efi vfat umask=0077 0 1" | sudo tee -a /etc/fstab
```

- `blkid` reads the partition's UUID, so the entry does not depend on `/dev/sda6` staying the same name.
- `umask=0077` keeps the EFI partition readable only by root.
- Later `grub` and kernel updates keep working because `/boot/efi` is always mounted.

**Outcome of Step 2:** the disk now has a classic BIOS-style boot path (if the VM had one) **and** a UEFI boot path, so the image can start on either type of laptop.

---

## Step 3 – Snapshot and USB preparation

1. **Check the VirtualBox snapshot.** Before you capture, make sure you have a snapshot of the VM. If anything in Step 2 went wrong, you can roll back.
2. **Prepare two USB drives:**

| Drive | Size | Format | Role |
|---|---|---|---|
| Boot drive | 8 GB | Created with Rufus or balenaEtcher from the Rescuezilla ISO | Starts Rescuezilla on the laptop |
| Storage drive | 64 GB | exFAT or NTFS | Holds the captured disk image |

---

## Step 4 – Capture the VM image with Rescuezilla

**Goal:** Save the VM's whole disk as an image file on the 64 GB USB.

1. **Give the VM access to the storage USB.**
   - In VirtualBox: *VM Settings → USB → enable **USB 3.0 (xHCI) Controller***.
   - Add a **USB device filter** for the 64 GB drive (or use *Devices → USB* while the VM runs) so the drive is passed through to the VM.
2. **Boot the VM into Rescuezilla** (live environment).
3. In Rescuezilla choose **Backup**:
   - Source: the VM's virtual disk (for example `sda`)
   - Partitions: select all, including the new EFI partition
   - Destination: the 64 GB USB drive
   - Give the image a clear name, for example `ubuntu-vm-v2p-image`
4. Wait for the backup to finish, then **shut down the VM** and safely remove the USB.

---

## Step 5 – Restore the image to the laptop

> **Warning:** Restoring **erases the laptop's target disk**. Back up anything you need first, and double-check the disk you select.

1. Plug in **both** USB drives (Rescuezilla boot USB and image storage USB).
2. Open the laptop's one-time boot menu (usually `F12`, `F10`, `Esc` or `F2`, depending on brand) and boot from the **Rescuezilla USB**.
3. Choose **Restore**:
   - Source image: the folder on the 64 GB USB
   - Destination disk: the laptop's **internal disk**
   - Restore the partition table as well, so the EFI partition is included
4. Confirm and wait for completion.
5. Reboot, remove both USB drives, and boot from the internal disk. If the laptop does not start Ubuntu automatically, pick the entry from the one-time boot menu.

---

## Post-migration checklist

- [ ] Ubuntu boots to the login screen
- [ ] **Network works.** If not, check the interface name with `ip a` and update `/etc/netplan/*.yaml`, then run `sudo netplan apply`.
- [ ] **Wi-Fi and Bluetooth work** (this is why `linux-firmware` was installed)
- [ ] **GPU driver.** For NVIDIA graphics: `sudo ubuntu-drivers autoinstall`
- [ ] **Remove VirtualBox leftovers.** Uninstall guest additions: `sudo apt remove virtualbox-guest-*`
- [ ] Check disk size: if the laptop disk is bigger, the extra space is unallocated. Extend the root partition and filesystem (for example with GParted, or `growpart` + `resize2fs`).
- [ ] `lsblk` and `cat /etc/fstab` look correct
- [ ] Run `sudo update-grub` once more on the laptop

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Laptop does not see Ubuntu in the boot menu | Enter firmware settings and make sure UEFI boot is enabled. The `--removable` install should appear as the internal disk or "UEFI: disk name". |
| Secure Boot blocks the boot | Temporarily disable Secure Boot in firmware, boot Ubuntu, then re-install the signed GRUB packages and re-enable it. |
| Boots to a `grub>` prompt | Boot a live Ubuntu USB, mount the root and EFI partitions, `chroot`, and run `grub-install` and `update-grub` again. |
| Boot hangs waiting for a device | A mount in `/etc/fstab` does not exist on the laptop. Remove it or add `nofail`. |
| No network after boot | Interface name changed. Update the netplan file as shown in the checklist. |
| Rescuezilla cannot see the VM disk or USB | Check that the VirtualBox Extension Pack is installed and the USB 3.0 (xHCI) controller and filter are enabled. |
| Image will not fit on the USB | Use a larger drive, or shrink the VM's partitions before the backup. |

---

## Important notes

- **Adapt the numbers.** Device names (`/dev/sda`), partition numbers and sizes in this guide belong to one specific VM. Always confirm yours with `lsblk` and `parted -l`.
- **Test before wiping.** If possible, try the restore on a spare disk or laptop first.
- **Keep the image.** The file on the 64 GB USB is a full backup of your VM, which is also useful for disaster recovery.
- **Use at your own risk.** Disk operations can destroy data if the wrong disk is selected.

---

## Tools used

- [VirtualBox](https://www.virtualbox.org/) – hypervisor running the source VM
- [Rescuezilla](https://rescuezilla.com/) – disk imaging and restore
- [Rufus](https://rufus.ie/) / [balenaEtcher](https://etcher.balena.io/) – make bootable USB drives
- GRUB, `parted`, `efibootmgr`, `dosfstools` – bootloader and partition tools
