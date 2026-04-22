# Boot Repair Guide

Fix boot issues for Windows and Linux systems.

---

## Boot Problem Symptoms

- "Operating system not found"
- "BOOTMGR is missing"
- "No bootable device"
- "Error: file '/grub/i386-pc/normal.mod' not found"
- Black screen with cursor
- Infinite boot loop

---

## Windows Boot Repair

### Windows Recovery Environment

**Access via:**
1. Windows installation USB → Repair your computer
2. Settings → Update & Security → Recovery → Advanced startup
3. Boot menu (F8/F12 during startup)

---

### Bootrec Commands

```cmd
# From Windows Recovery command prompt
bootrec /fixmbr
bootrec /fixboot
bootrec /scanos
bootrec /rebuildbcd
```

| Command | Purpose |
|---------|---------|
| `/fixmbr` | Repair Master Boot Record |
| `/fixboot` | Write new boot sector |
| `/scanos` | Scan for Windows installations |
| `/rebuildbcd` | Rebuild Boot Configuration Data |

### Detailed Steps

```cmd
# 1. Check Windows installations
bootrec /scanos

# 2. Rebuild BCD
bootrec /rebuildbcd
# When asked "Add installation to boot list?" → Y

# 3. If that fails, manual BCD rebuild:
bcdedit /export C:\BCD_Backup
attrib C:\boot\bcd -h -r -s
ren C:\boot\bcd bcd.old
bootrec /rebuildbcd
```

### UEFI Specific Commands

```cmd
# For UEFI systems
bootrec /fixmbr
bootrec /fixboot

# If "Access denied" on fixboot:
diskpart
list disk
select disk 0
list volume
select volume X  (EFI partition, usually FAT32)
assign letter=S:
exit

bootrec /fixboot
bcdboot C:\Windows /s S: /f UEFI
```

### BCDBoot - Nuclear Option

```cmd
# Completely rebuild Windows boot files
bcdboot C:\Windows

# UEFI specific
bcdboot C:\Windows /s S: /f UEFI

# BIOS/Legacy
bcdboot C:\Windows /s C: /f BIOS
```

---

## Disk and File System Checks

```cmd
# Check disk
chkdsk C: /f
chkdsk C: /f /r
chkdsk C: /f /r /x

# System file checker
sfc /scannow /offbootdir=C:\ /offwindir=C:\Windows

# DISM repair
DISM /Image:C:\ /Cleanup-Image /RestoreHealth
```

---

## Linux Boot Repair

### GRUB Recovery

**From Live USB:**

```bash
# 1. Boot BlackArch/Ubuntu Live USB

# 2. Identify Linux partition
sudo fdisk -l
# Look for Linux filesystem (ext4, btrfs, etc.)

# 3. Mount partitions
sudo mount /dev/sda2 /mnt  # Root partition
sudo mount /dev/sda1 /mnt/boot/efi  # EFI partition (if UEFI)
sudo mount /dev/sda3 /mnt/boot      # Separate boot partition (if exists)

# 4. Bind system directories
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys

# 5. Chroot into system
sudo chroot /mnt

# 6. Reinstall GRUB
# For BIOS/MBR:
grub-install /dev/sda

# For UEFI:
grub-install --target=x86_64-efi --efi-directory=/boot/efi --bootloader-id=GRUB

# 7. Update GRUB config
grub-mkconfig -o /boot/grub/grub.cfg

# 8. Exit and reboot
exit
sudo umount -R /mnt
reboot
```

---

### GRUB Rescue Prompt

If you get `grub rescue>` prompt:

```grub
# Find GRUB modules
ls
# Should show (hd0), (hd0,gpt1), (hd0,gpt2), etc.

# Find boot partition
ls (hd0,gpt2)/boot/grub

# Set prefix
set prefix=(hd0,gpt2)/boot/grub

# Load normal module
insmod normal
normal

# Now you should see GRUB menu
# Or manually boot:
linux /boot/vmlinuz-linux root=/dev/sda2
initrd /boot/initramfs-linux.img
boot
```

---

### EFI Boot Manager Repair

```bash
# From Live USB
sudo mount /dev/sda1 /mnt/efi  # EFI partition

# List current entries
efibootmgr -v

# Create new entry
sudo efibootmgr -c -d /dev/sda -p 1 -L "BlackArch" -l "\EFI\blackarch\grubx64.efi"

# Remove broken entry
sudo efibootmgr -b 0003 -B  # Remove bootnum 0003

# Change boot order
sudo efibootmgr -o 0001,0002,0003
```

---

## Dual Boot Repair

### Windows Broke GRUB

```bash
# From Linux Live USB
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot/efi
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt

# Reinstall GRUB
grub-install /dev/sda
update-grub
```

### Add Windows to GRUB

```bash
# If os-prober doesn't find Windows
sudo nano /etc/grub.d/40_custom

# Add:
menuentry "Windows 10" {
    insmod part_gpt
    insmod fat
    insmod chain
    search --fs-uuid --set=root YOUR-EFI-PARTITION-UUID
    chainloader /EFI/Microsoft/Boot/bootmgfw.efi
}

# Update
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

---

## BIOS/UEFI Settings

### Important Settings to Check

| Setting | Value For | Notes |
|---------|-----------|-------|
| Boot Mode | UEFI for modern, Legacy for old | Match installation |
| Secure Boot | OFF for Linux Live USBs | Can re-enable after |
| Fast Boot | OFF for troubleshooting | Skips detection |
| CSM | OFF for pure UEFI | Compatibility mode |
| SATA Mode | AHCI (not RAID/IDE) | Better compatibility |

---

## Boot from Wrong Device

### Fix Boot Order

```bash
# In BIOS/UEFI:
# 1. Check Boot Priority
# 2. Move correct drive to top
# 3. If multiple drives, ensure boot drive is first

# Or from EFI shell
bcfg boot dump
bcfg boot mv 3 0  # Move boot entry 3 to position 0
```

---

## Repair from Disk Images

### Mount and Fix Image

```bash
# Mount disk image
sudo losetup -fP /mnt/backup/disk.img

# Mount partitions
sudo mount /dev/loop0p2 /mnt/fix
sudo mount /dev/loop0p1 /mnt/fix/boot/efi

# Chroot and repair
sudo mount --bind /dev /mnt/fix/dev
sudo mount --bind /proc /mnt/fix/proc
sudo chroot /mnt/fix

# Now run grub-install, update-grub, etc.
```

---

## Common Errors

### "error: no such partition"

```bash
# Partition UUID changed
# Boot from Live USB and update GRUB:
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot/efi
sudo chroot /mnt
update-grub  # Regenerates with correct UUIDs
```

### "error: file '/boot/grub/i386-pc/normal.mod' not found"

```bash
# GRUB files corrupted or wrong architecture
# Reinstall GRUB completely:
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot/efi
sudo chroot /mnt
grub-install /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

### "bootmgr is compressed"

```cmd
# From Windows Recovery
expand C:\bootmgr C:\bootmgr.new
del C:\bootmgr
ren C:\bootmgr.new bootmgr

# Or disable compression:
compact /u /s:C:\
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

---

## Emergency Boot Methods

### Chainloading

```grub
# Boot Windows from GRUB
insmod chain
insmod ntfs
set root=(hd0,1)
chainloader +1
boot
```

### Super Grub2 Disk

```bash
# Download Super Grub2 Disk ISO
# Write to USB and boot
# Auto-detects all OSes
# Can boot even with broken GRUB
```

### Rescatux (GUI Repair)

```bash
# Boot Rescatux Live CD
# Graphical boot repair for Linux/Windows
# Automatic GRUB restoration
```

---

## Quick Boot Repair Checklist

```bash
# 1. Check disk is detected
lsblk
fdisk -l

# 2. Check partition table
gdisk -l /dev/sda

# 3. Check filesystems
lsblk -f

# 4. Check boot flags
parted /dev/sda print
# Look for 'boot' flag on EFI partition

# 5. Check EFI partition contents
ls /boot/efi/EFI/
# Should see: BOOT, Microsoft, (your distro)

# 6. Verify GRUB installed
ls /boot/grub/
ls /boot/efi/EFI/(distro)/
```

---

## Windows Boot Files Reference

| File | Location | Purpose |
|------|----------|---------|
| bootmgr | C:\ | Boot manager |
| BCD | C:\Boot\ | Boot configuration |
| winload.exe | C:\Windows\System32\ | Windows loader |
| bootmgfw.efi | EFI\Microsoft\Boot\ | UEFI boot manager |
| bootmgr.efi | EFI\Boot\ | Fallback EFI boot |

---

## Linux Boot Files Reference

| File | Location | Purpose |
|------|----------|---------|
| vmlinuz | /boot/ | Kernel |
| initramfs | /boot/ | Initial RAM disk |
| grub.cfg | /boot/grub/ | GRUB configuration |
| grubx64.efi | /boot/efi/EFI/(distro)/ | UEFI GRUB |
| fstab | /etc/ | Filesystem mounts |
