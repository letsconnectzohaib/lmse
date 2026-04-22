# Linux Data Recovery Guide

Complete guide for recovering data from corrupted Windows drives using Linux Live USB.

---

## Boot into Linux Live USB

1. Create bootable USB with BlackArch/Ubuntu/Kali
2. Boot corrupted PC from USB
3. Open terminal

---

## Step 1: Detect Disks

```bash
lsblk
```

**Look for:**
- `sda`, `sdb` (SATA drives)
- `nvme0n1` (NVMe SSDs)
- Partitions like `sda1`, `sda2`, `nvme0n1p1`

---

## Step 2: Check Filesystem Type

```bash
lsblk -f
```

| FSTYPE | Meaning | Action |
|--------|---------|--------|
| `ntfs` | Normal Windows | Mount directly |
| `crypto_LUKS` | Linux encryption | Need password |
| `unknown` / blank | Damaged or BitLocker | See BitLocker guide |
| `BitLocker` | Windows encryption | Need recovery key |
| `vfat` | EFI boot partition | Skip |

---

## Step 3: Try Mounting NTFS Partition

```bash
# Create mount point
sudo mkdir -p /mnt/windows

# Try mounting (replace sda2 with your partition)
sudo mount /dev/sda2 /mnt/windows

# Check if successful
ls /mnt/windows
```

**If you see `Users/` folder = SUCCESS** ✅

---

## Step 4: If Mount Fails

### Fix NTFS errors:

```bash
sudo ntfsfix /dev/sda2
sudo mount /dev/sda2 /mnt/windows
```

### Check for hardware errors:

```bash
dmesg | grep -i error
smartctl -a /dev/sda
```

---

## Step 5: Copy Your Data

```bash
# Mount external USB for backup
sudo mkdir -p /mnt/backup
sudo mount /dev/sdb1 /mnt/backup  # replace with your USB

# Copy important folders
cp -r /mnt/windows/Users/YourName/Documents /mnt/backup/
cp -r /mnt/windows/Users/YourName/Downloads /mnt/backup/
cp -r /mnt/windows/Users/YourName/Desktop /mnt/backup/
cp -r /mnt/windows/Users/YourName/Pictures /mnt/backup/

# Unmount when done
sudo umount /mnt/windows
sudo umount /mnt/backup
```

---

## Tools for Corrupted Filesystems

| Tool | Use Case | Install |
|------|----------|---------|
| `ntfsfix` | Repair NTFS | Built-in (ntfs-3g) |
| `ntfs-3g` | Mount NTFS | `sudo pacman -S ntfs-3g` |
| `testdisk` | Recover partitions | `sudo pacman -S testdisk` |
| `photorec` | Recover raw files | Part of testdisk |
| `ddrescue` | Create disk image | `sudo pacman -S ddrescue` |

---

## Creating Disk Image (Safest Method)

```bash
# Create image of corrupted drive
sudo ddrescue /dev/sda /mnt/backup/disk-image.img /mnt/backup/logfile

# Mount image to work on copy
sudo losetup -fP /mnt/backup/disk-image.img
sudo mount /dev/loop0p2 /mnt/windows
```

---

## Quick Command Reference

```bash
# List all disks
fdisk -l

# Check partition details
file -s /dev/sda2

# Check first bytes (verify BitLocker)
xxd -l 256 /dev/sda2 | head -5

# Mount with read-only (safer)
sudo mount -o ro /dev/sda2 /mnt/windows

# Force mount damaged NTFS
sudo mount -t ntfs-3g -o force,rw /dev/sda2 /mnt/windows
```
