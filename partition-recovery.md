# Partition Recovery Guide

Recover lost, deleted, or corrupted partitions using various tools.

---

## Symptoms of Lost Partitions

- Drive shows as "unallocated" in disk management
- `lsblk` shows disk but no partitions
- "Invalid partition table" errors
- Accidental deletion with `fdisk` or `gparted`

---

## TestDisk (Primary Tool)

### Install

```bash
sudo pacman -S testdisk
```

### Basic Partition Recovery

```bash
sudo testdisk /dev/sda
```

**Interactive Steps:**

1. **Select disk**: Choose `/dev/sda` (or your drive)
2. **Select partition table type**:
   - `[Intel ]` - Most Windows PCs (MBR)
   - `[EFI GPT]` - Modern systems (GPT)
   - `[None ]` - Raw recovery
3. **Select analysis**:
   - `[Analyse]` - Quick/deep search
   - `[Advanced]` - Filesystem repair
   - `[Geometry]` - Change disk geometry
4. **Quick Search**: Finds recently deleted partitions
5. **Deeper Search**: Finds older/deleted partitions (slow)
6. **Select found partitions**: Press `P` to list files (verify it's correct)
7. **Write changes**: Select `[Write]` to restore partition table

---

## TestDisk: Step-by-Step

### 1. Analyze Current Structure

```bash
sudo testdisk /dev/sda
# > [Proceed] > [Intel] or [EFI GPT] > [Analyse] > [Quick Search]
```

### 2. Find Lost Partitions

```bash
# In TestDisk:
# Quick Search finds: Existing + recently deleted partitions
# Deeper Search finds: Older deleted partitions

# Use arrow keys to navigate, Enter to select
# Status indicators:
# * = Primary bootable
# P = Primary
# L = Logical
# D = Deleted
```

### 3. Verify Before Writing

```bash
# For each found partition:
# Press 'P' to list files
# Verify it's the right partition
# If correct: Press 'Enter' to change D(deleted) to L/P
```

### 4. Write Partition Table

```bash
# After selecting all partitions to recover:
# > [Write] > Confirm > Reboot
```

---

## GParted Recovery

### For GPT Partitions

```bash
sudo pacman -S gparted
sudo gparted

# GUI options:
# 1. Select disk from dropdown
# 2. Device → Attempt Data Rescue
# 3. Follow wizard to find lost partitions
```

---

## FDisk / GDisk for Partition Repair

### Fix Corrupted MBR

```bash
# Backup current MBR first
sudo dd if=/dev/sda of=/mnt/backup/mbr-backup.bin bs=512 count=1

# Restore standard MBR
sudo fdisk /dev/sda
# > o  (create new empty DOS partition table)
# > w  (write and exit)

# Then use TestDisk to recover partitions
```

### Fix Corrupted GPT

```bash
# Use gdisk for GPT disks
sudo gdisk /dev/sda

# Recovery options:
# > r  (recovery and transformation)
# > b  (use backup GPT header)
# > c  (load backup from file)
# > d  (use main GPT header)
# > w  (write changes)
```

---

## Photorec for Partitionless Recovery

When partition table is too damaged:

```bash
sudo photorec /dev/sda

# Options:
# [Search] - Start at beginning of disk
# [Options] - File types to recover
# [File Opt] - Select specific file extensions
# [Quit]

# Recovery destination: Choose different disk!
```

---

## Common Scenarios

### Scenario 1: Accidentally Deleted Partition

```bash
# STOP - Don't write anything to disk!

# Use TestDisk immediately
sudo testdisk /dev/sda

# > [Proceed]
# > [Intel/EFI GPT] (select correct type)
# > [Analyse]
# > [Deeper Search]
# Find your deleted partition (marked D)
# Press Enter to change status from D to L/P
# [Write]
# Reboot
```

### Scenario 2: Partition Shows as RAW/Unknown

```bash
# Check filesystem type
sudo file -s /dev/sda2

# If shows "data" instead of filesystem:
# Likely corrupted superblock

# For NTFS:
sudo ntfsfix /dev/sda2

# For ext4:
sudo fsck.ext4 -b 32768 /dev/sda2  # Try backup superblock

# Use TestDisk Advanced menu:
sudo testdisk /dev/sda
# > [Advanced] > Select partition > [Type] > [Boot]
```

### Scenario 3: Drive Shows Wrong Size

```bash
# Wrong geometry detected
sudo testdisk /dev/sda
# > [Geometry]
# Set correct:
# - Cylinders
# - Heads
# - Sectors per track

# Usually: 255 heads, 63 sectors for HDDs
```

---

## Advanced TestDisk Features

### Rebuild Boot Sector (NTFS/FAT)

```bash
sudo testdisk /dev/sda
# > [Advanced] > Select partition > [Boot]
# Options:
# - [Rebuild BS] - Rebuild boot sector
# - [Repair MFT] - Repair NTFS Master File Table
# - [List] - List files even if damaged
```

### Copy Files from Damaged Partition

```bash
sudo testdisk /dev/sda
# > [Advanced] > Select partition > [List]
# Navigate with arrow keys
# Select files/folders with ':'
# Press 'C' to copy
# Choose destination directory
# Press 'c' to confirm copy
```

---

## Command-Line Partition Tools

### Parted Rescue Mode

```bash
sudo parted /dev/sda

# Check partitions
print

# Rescue mode (find partitions between start-end)
rescue 0 500GB

# Or scan entire disk
rescue 0 100%
```

### Using Sfdisk

```bash
# Backup partition table
sudo sfdisk -d /dev/sda > /mnt/backup/partition-table.txt

# Restore partition table
sudo sfdisk /dev/sda < /mnt/backup/partition-table.txt
```

---

## Error Fixes

### "Partition table entries are not in disk order"

```bash
# Harmless warning, but can fix:
sudo fdisk /dev/sda
# > x (extra)
# > f (fix order)
# > r (return)
# > w (write)
```

### "GPT PMBR size mismatch"

```bash
# Fix with gdisk
sudo gdisk /dev/sda
# > w (write - will fix automatically)
# Confirm: Y
```

### "Superblock corrupt" (ext2/3/4)

```bash
# Find backup superblocks
sudo dumpe2fs /dev/sda2 | grep -i superblock

# Repair with backup
sudo e2fsck -b 32768 /dev/sda2
# or
sudo e2fsck -b 98304 /dev/sda2
```

---

## Preventing Future Partition Loss

### Backup Partition Tables

```bash
# MBR backup
sudo dd if=/dev/sda of=/mnt/backup/sda-mbr.bin bs=512 count=1

# GPT backup
sudo sgdisk -b /mnt/backup/sda-gpt.gpt /dev/sda

# Full sfdisk dump
sudo sfdisk -d /dev/sda > /mnt/backup/sda-partitions.txt
```

### Restore When Needed

```bash
# Restore MBR
sudo dd if=/mnt/backup/sda-mbr.bin of=/dev/sda bs=512 count=1

# Restore GPT
sudo sgdisk -l /mnt/backup/sda-gpt.gpt /dev/sda

# Restore from sfdisk
sudo sfdisk /dev/sda < /mnt/backup/sda-partitions.txt
```

---

## Quick Reference

| Tool | Use Case |
|------|----------|
| TestDisk | Deleted partitions, rebuild boot sector |
| GParted | GUI partition management, data rescue |
| FDisk | MBR partition table editing |
| GDisk | GPT partition table editing |
| Parted | Scriptable partition operations |
| PhotoRec | File recovery when partition lost |

---

## Emergency Steps

```bash
# 1. STOP all writes to disk immediately

# 2. Create disk image if possible
sudo ddrescue /dev/sda /mnt/backup/emergency.img /mnt/backup/emergency.log

# 3. Work on image, not original
sudo losetup -fP /mnt/backup/emergency.img
sudo testdisk /dev/loop0

# 4. Recover partitions

# 5. When successful, write changes
