# Filesystem Repair Guide

Repair corrupted filesystems on Windows (NTFS), Linux (ext4), and other formats.

---

## NTFS Repair (Windows Drives)

### ntfsfix - Quick Fix

```bash
# Install
sudo pacman -S ntfs-3g

# Basic repair
sudo ntfsfix /dev/sda2

# Clear dirty flag (fast boot/hybrid shutdown issues)
sudo ntfsfix -d /dev/sda2

# Clear Windows hibernation file
sudo ntfsfix -b /dev/sda2

# Both clear flags
sudo ntfsfix -bd /dev/sda2
```

### Force Mount with Repair

```bash
# Mount with remove_hiberfile (clears Windows fast boot)
sudo mount -t ntfs-3g -o remove_hiberfile,rw /dev/sda2 /mnt/windows

# Or force mount (risks data loss)
sudo mount -t ntfs-3g -o force,rw /dev/sda2 /mnt/windows

# Read-only safe mount
sudo mount -t ntfs-3g -o ro /dev/sda2 /mnt/windows
```

### chkdsk from Windows Recovery

```cmd
# From Windows Recovery USB command prompt
chkdsk C: /f
chkdsk C: /f /r
chkdsk C: /f /r /x
```

| Parameter | Function |
|-----------|----------|
| `/f` | Fix errors on disk |
| `/r` | Recover bad sectors |
| `/x` | Force dismount first |
| `/v` | Verbose mode |
| `/scan` | Online scan (Windows 10+) |

---

## ext4 Repair (Linux Drives)

### e2fsck - ext2/3/4 Filesystem Check

```bash
# Unmount first (critical!)
sudo umount /dev/sda2

# Basic check
sudo e2fsck /dev/sda2

# Auto-fix without prompting
sudo e2fsck -p /dev/sda2

# Force check even if "clean"
sudo e2fsck -f /dev/sda2

# Verbose with progress
sudo e2fsck -v /dev/sda2
```

### Repair with Backup Superblock

```bash
# Find backup superblocks
sudo dumpe2fs /dev/sda2 | grep -i superblock
# Output: Backup superblock at 32768, 98304, 163840, 229376, 294912...

# Repair using backup superblock
sudo e2fsck -b 32768 /dev/sda2

# If that fails, try next
sudo e2fsck -b 98304 /dev/sda2
```

### Rebuild ext4 Journal

```bash
# For severe corruption
sudo e2fsck -y /dev/sda2

# Force journal replay
sudo e2fsck -j /dev/sda2
```

---

## FAT32 / exFAT Repair

### fsck.vfat (FAT12/16/32)

```bash
# Install
sudo pacman -S dosfstools

# Check and repair
sudo fsck.vfat /dev/sdb1

# Auto-repair
sudo fsck.vfat -a /dev/sdb1

# Interactive repair
sudo fsck.vfat -r /dev/sdb1

# Verbose
sudo fsck.vfat -v /dev/sdb1
```

### fsck.exfat

```bash
# Install
sudo pacman -S exfat-utils

# Check exFAT
sudo fsck.exfat /dev/sdb1

# Repair
sudo fsck.exfat -a /dev/sdb1
```

---

## Common Repair Scenarios

### Windows Won't Boot - "Unmountable Boot Volume"

```bash
# From Linux Live USB
sudo ntfsfix /dev/sda2
sudo ntfsfix -b /dev/sda2  # Clear hibernation

# Mount and check files
sudo mount -t ntfs-3g -o remove_hiberfile /dev/sda2 /mnt/windows
ls /mnt/windows/Windows/System32
```

### "Filesystem is dirty" Error

```bash
# Check why it's dirty
sudo ntfscluster -f /dev/sda2

# Force clean (use with caution)
sudo ntfsfix -d /dev/sda2

# Better: Run from Windows
# chkdsk C: /f /r
```

### Corrupted MFT (Master File Table)

```bash
# Use TestDisk to rebuild MFT
sudo testdisk /dev/sda
# > [Advanced] > Select NTFS partition > [Boot] > [Repair MFT]
```

### Lost+Found Recovery (ext4)

```bash
# After e2fsck, check lost+found
sudo ls -la /mnt/linux/lost+found/

# Identify files
sudo file /mnt/linux/lost+found/*

# Move recovered files
sudo mv /mnt/linux/lost+found/file123 /mnt/recovered/
```

---

## Btrfs Repair

```bash
# Check filesystem
sudo btrfs check /dev/sda2

# Repair (use with caution)
sudo btrfs check --repair /dev/sda2

# Scrub (verify and auto-heal)
sudo btrfs scrub start /mnt/btrfs

# Cancel scrub
sudo btrfs scrub cancel /mnt/btrfs
```

---

## XFS Repair

```bash
# Unmount first
sudo umount /dev/sda2

# Check (read-only)
sudo xfs_repair -n /dev/sda2

# Repair
sudo xfs_repair /dev/sda2

# Force repair (dangerous)
sudo xfs_repair -L /dev/sda2  # Zeroes log, may lose data
```

---

## ReFS (Windows) Limited Support

```bash
# Linux has read-only support
sudo modprobe refs
sudo mount -t refs -o ro /dev/sda2 /mnt/windows

# For repair, must use Windows:
# chkdsk C: /f
```

---

## Advanced Repair Techniques

### Wipe and Recreate Filesystem (Last Resort)

```bash
# WARNING: DESTROYS ALL DATA

# Backup what you can first
sudo ddrescue /dev/sda2 /mnt/backup/partition.img /mnt/backup/logfile

# Wipe filesystem signature
sudo wipefs -a /dev/sda2

# Recreate NTFS
sudo mkfs.ntfs -f /dev/sda2

# Recreate ext4
sudo mkfs.ext4 /dev/sda2

# Then use photorec on the backup image
```

### Zero Bad Sectors

```bash
# Find bad sectors
sudo badblocks -sv /dev/sda

# Write zeros to bad sectors (forces reallocation)
sudo badblocks -wsv /dev/sda
# WARNING: DESTRUCTIVE - wipes all data
```

---

## Repair from Disk Images

### Mount and Repair Image

```bash
# Setup loop
sudo losetup -fP /mnt/backup/disk.img

# Check filesystem
sudo ntfsfix /dev/loop0p2
# or
sudo e2fsck /dev/loop0p2

# Mount repaired
sudo mount /dev/loop0p2 /mnt/fixed
```

---

## Error Reference

### NTFS Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| "dirty flag set" | Unclean shutdown | `ntfsfix -d` |
| "hiberfil present" | Windows hibernated | `ntfsfix -b` or `mount -o remove_hiberfile` |
| "MFT corrupt" | Master File Table damaged | TestDisk Advanced > Repair MFT |
| "bitmap corrupt" | Allocation bitmap error | `chkdsk /f` from Windows |
| "$Bitmap truncated" | Bitmap file truncated | `ntfsfix` then `chkdsk` |

### ext4 Errors

| Error | Meaning | Fix |
|-------|---------|-----|
| "Superblock corrupt" | Primary superblock bad | `e2fsck -b 32768` |
| "Journal corrupt" | Journal replay failed | `e2fsck -j` |
| "Group descriptors bad" | Group descriptor corruption | `e2fsck -y` |
| "Inode table bad" | Inode table damaged | `e2fsck -f` |
| "Orphaned inodes" | Lost inodes found | Check lost+found after repair |

---

## Filesystem Check Schedule

### Force Check on Next Boot (Linux)

```bash
# Mark for check
sudo tune2fs -C 40 /dev/sda2

# Or set force flag
sudo tune2fs -c 0 /dev/sda2  # Disable count-based check
```

---

## Quick Commands

```bash
# Check all mounted filesystems
df -h

# Check filesystem type
lsblk -f

# Force check unmounted NTFS
sudo ntfsfix -b -d /dev/sda2

# Safe mount read-only
sudo mount -o ro /dev/sda2 /mnt/safe

# Check for I/O errors
dmesg | grep -i "error\|fail\|corrupt"
```
