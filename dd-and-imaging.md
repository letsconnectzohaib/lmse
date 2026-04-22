# Disk Imaging with DD and DDRescue

Complete guide for creating disk images and working with them safely.

---

## Why Create Disk Images?

- **Work on copy, not original** (safer)
- **Drive is failing** (minimize reads)
- **Forensic analysis** (preserve evidence)
- **Multiple recovery attempts** (start fresh each time)

---

## DD (Basic Imaging)

### Simple Full Disk Image

```bash
# Create exact copy of entire disk
sudo dd if=/dev/sda of=/mnt/backup/sda-image.img bs=4M status=progress
```

### Image Specific Partition

```bash
# Image just one partition
sudo dd if=/dev/sda2 of=/mnt/backup/windows-partition.img bs=4M status=progress
```

### Compressed Image (Save Space)

```bash
# Compress on the fly
sudo dd if=/dev/sda bs=4M status=progress | gzip > /mnt/backup/sda-image.img.gz

# Or with pigz (parallel, faster)
sudo dd if=/dev/sda bs=4M status=progress | pigz > /mnt/backup/sda-image.img.gz
```

### Split Large Images

```bash
# Split into 4GB chunks (FAT32 compatible)
sudo dd if=/dev/sda bs=4M status=progress | split -b 4G - /mnt/backup/sda-image.img.part-

# Reassemble later:
cat /mnt/backup/sda-image.img.part-* > sda-image.img
```

---

## DDRescue (For Failing Drives)

**Always use ddrescue for damaged/failing drives!**

### Basic DDRescue

```bash
# Install
sudo pacman -S ddrescue

# First pass: copy good sectors quickly
sudo ddrescue -n /dev/sda /mnt/backup/sda-rescue.img /mnt/backup/rescue.log

# Second pass: retry bad sectors
sudo ddrescue -d -r3 /dev/sda /mnt/backup/sda-rescue.img /mnt/backup/rescue.log
```

### Advanced DDRescue Strategy

```bash
# Phase 1: Quick pass, skip errors
sudo ddrescue -n -b4096 /dev/sda /mnt/backup/disk.img /mnt/backup/logfile

# Phase 2: Retry errors 3 times
sudo ddrescue -d -r3 -b4096 /dev/sda /mnt/backup/disk.img /mnt/backup/logfile

# Phase 3: Direct access, no kernel cache
sudo ddrescue -d -r3 -b512 /dev/sda /mnt/backup/disk.img /mnt/backup/logfile
```

### DDRescue Options Explained

| Option | Meaning |
|--------|---------|
| `-n` | No scrape (skip bad sectors initially) |
| `-d` | Direct disk access (bypass cache) |
| `-r3` | Retry bad sectors 3 times |
| `-b4096` | Block size 4KB (adjust for drive) |
| `-v` | Verbose output |

---

## Cloning to Another Disk

### Direct Disk Clone

```bash
# Clone sda to sdb (destroys sdb contents!)
sudo dd if=/dev/sda of=/dev/sdb bs=4M status=progress conv=noerror,sync

# Safer with ddrescue:
sudo ddrescue /dev/sda /dev/sdb /mnt/backup/clone.log
```

### Partition to Partition

```bash
# Clone single partition
sudo dd if=/dev/sda2 of=/dev/sdb1 bs=4M status=progress
```

---

## Mounting Disk Images

### Raw Image

```bash
# Setup loop device
sudo losetup -fP /mnt/backup/sda-image.img

# Find loop device
losetup -a

# Mount partition from image (loop0p1 = first partition)
sudo mount /dev/loop0p2 /mnt/image-mount

# Access files
ls /mnt/image-mount

# Cleanup when done
sudo umount /mnt/image-mount
sudo losetup -d /dev/loop0
```

### Compressed Image

```bash
# Mount compressed image directly
gunzip -c /mnt/backup/sda-image.img.gz | sudo losetup -fP /dev/stdin

# Or extract first:
gunzip /mnt/backup/sda-image.img.gz
# Then mount as raw image
```

---

## Selective Region Imaging

### Image Specific Sector Range

```bash
# Image sectors 1000-5000 (useful for partition recovery)
sudo dd if=/dev/sda of=/mnt/backup/partial.img bs=512 skip=1000 count=4000

# Or with ddrescue
sudo ddrescue -s 512000 -S 2048000 /dev/sda /mnt/backup/partial.img /mnt/backup/partial.log
# -s = start position (bytes)
# -S = max size (bytes)
```

---

## Image Verification

### Check Image Integrity

```bash
# Calculate checksum of source (optional - adds read stress)
sudo dd if=/dev/sda bs=4M | md5sum

# Checksum of image
md5sum /mnt/backup/sda-image.img

# Compare (should match if drive is good)
```

### Verify Image Mounts

```bash
# Test mount loop device
sudo losetup -fP /mnt/backup/sda-image.img
sudo file -s /dev/loop0p2

# Should show: "DOS/MBR boot sector, NTFS ..."
```

---

## Error Handling

### DD Errors and Fixes

#### "Input/output error"

```bash
# Use ddrescue instead
sudo ddrescue /dev/sda /mnt/backup/image.img /mnt/backup/logfile

# Or skip errors with dd
sudo dd if=/dev/sda of=/mnt/backup/image.img bs=4M conv=noerror,sync status=progress
```

#### "No space left on device"

```bash
# Check destination space
df -h /mnt/backup

# Compress on the fly
sudo dd if=/dev/sda bs=4M | xz -9e > /mnt/backup/image.img.xz

# Or use external drive with more space
sudo dd if=/dev/sda of=/mnt/external-drive/image.img bs=4M status=progress
```

#### "Read-only file system"

```bash
# Remount as read-write
sudo mount -o remount,rw /mnt/backup

# Or check filesystem
sudo fsck /dev/sdb1  # your backup partition
```

### DDRescue Log File Recovery

```bash
# If ddrescue interrupted, resume with same log
sudo ddrescue -r3 /dev/sda /mnt/backup/partial.img /mnt/backup/existing.log

# Check log status
sudo ddrescue -i /mnt/backup/logfile
```

---

## Forensic Imaging (Bit-Perfect)

### Using DCFLDD

```bash
# Install
sudo pacman -S dcfldd

# Forensic image with hashing
sudo dcfldd if=/dev/sda of=/mnt/backup/forensic.img bs=4M hash=md5,sha256 hashlog=/mnt/backup/forensic.hashes
```

### Using Guymager

```bash
# GUI forensic imager
sudo pacman -S guymager
sudo guymager

# Select drive -> Create Image -> Select format (E01 or raw)
```

---

## Working with Images

### Extract Specific Files from Image

```bash
# Mount and copy
sudo losetup -fP image.img
sudo mount /dev/loop0p2 /mnt/work

# Find and copy
find /mnt/work -name "*.docx" -exec cp {} /mnt/recovered/ \;

# Unmount
sudo umount /mnt/work
sudo losetup -d /dev/loop0
```

### Image to Virtual Machine

```bash
# Convert raw image to VDI (VirtualBox)
VBoxManage convertdd disk.img disk.vdi --format VDI

# Or to QCOW2 (QEMU/KVM)
qemu-img convert -f raw -O qcow2 disk.img disk.qcow2
```

---

## Quick Commands

```bash
# Fast disk status
ddrescue --status /mnt/backup/logfile

# Pause ddrescue (sends SIGUSR1)
kill -USR1 $(pgrep ddrescue)

# Monitor progress in another terminal
watch -n 1 'sudo kill -USR1 $(pgrep ^dd)'
```
