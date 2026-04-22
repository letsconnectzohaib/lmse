# Quick Command Cheat Sheet

Fast reference for emergency data recovery commands.

---

## Emergency Assessment (First 2 Minutes)

```bash
# 1. List drives
lsblk -f

# 2. Check filesystem
file -s /dev/sda2

# 3. Check health
smartctl -H /dev/sda

# 4. Check recent errors
dmesg | tail -20 | grep -i error
```

---

## Quick Mount Commands

```bash
# Normal Windows mount
sudo mount /dev/sda2 /mnt/windows

# Force mount (corrupted)
sudo mount -t ntfs-3g -o force,rw /dev/sda2 /mnt/windows

# Remove hibernation
sudo mount -o remove_hiberfile /dev/sda2 /mnt/windows

# Read-only (safest)
sudo mount -o ro /dev/sda2 /mnt/windows

# Mount image
sudo losetup -fP image.img
sudo mount /dev/loop0p2 /mnt/image
```

---

## Quick BitLocker Commands

```bash
# Install dislocker
sudo pacman -S dislocker

# Unlock with key
sudo dislocker /dev/sda2 -pXXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX -- /mnt/dislocker

# Unlock with password
sudo dislocker /dev/sda2 -u -- /mnt/dislocker

# Mount decrypted
sudo mount /mnt/dislocker/dislocker-file /mnt/windows
```

---

## Quick Imaging Commands

```bash
# Simple dd
sudo dd if=/dev/sda of=/mnt/backup/image.img bs=4M status=progress

# Compressed dd
dd if=/dev/sda bs=4M | gzip > image.img.gz

# Ddrescue (for bad drives)
sudo ddrescue -n /dev/sda image.img logfile
sudo ddrescue -r3 /dev/sda image.img logfile

# Resume ddrescue
sudo ddrescue -r3 /dev/sda image.img existing.log

# Mount image
sudo losetup -fP image.img
mount /dev/loop0p2 /mnt/work
```

---

## Quick Repair Commands

```bash
# NTFS fix
sudo ntfsfix /dev/sda2
sudo ntfsfix -b -d /dev/sda2

# ext4 fix
sudo e2fsck -y /dev/sda2
sudo e2fsck -b 32768 /dev/sda2  # backup superblock

# FAT fix
sudo fsck.vfat -a /dev/sda1

# XFS fix
sudo xfs_repair /dev/sda2

# Btrfs fix
sudo btrfs check --repair /dev/sda2
```

---

## Quick File Recovery

```bash
# TestDisk
sudo testdisk /dev/sda
# > [Analyse] > [Quick Search] > [Write]

# PhotoRec
sudo photorec /dev/sda2

# Foremost
sudo foremost -i /dev/sda2 -o /mnt/recovery/

# Scalpel
sudo scalpel /dev/sda2 -o /mnt/recovery/
```

---

## Quick Copy Commands

```bash
# Basic copy
cp -r /mnt/windows/Users/Name/Documents /mnt/backup/

# Preserve permissions
cp -a /mnt/source /mnt/backup/

# Rsync (with resume)
rsync -avh --progress /source/ /dest/
rsync -avzP /source/ user@host:/dest/

# Secure copy
scp -r /mnt/windows/Users/ user@host:/backup/
```

---

## Quick Boot Repair

```bash
# Windows Recovery (from USB):
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
bcdboot C:\Windows /f UEFI
chkdsk C: /f /r

# BitLocker check
manage-bde -status
manage-bde -unlock C: -rp KEY
```

---

## Quick Partition Commands

```bash
# List partitions
fdisk -l /dev/sda
gdisk -l /dev/sda
parted /dev/sda print

# Create mount points
sudo mkdir -p /mnt/{work,backup,recovery}

# Backup partition table
sfdisk -d /dev/sda > partitions.txt
dd if=/dev/sda of=mbr.bin bs=512 count=1

# Restore partition table
sfdisk /dev/sda < partitions.txt
```

---

## Quick Network Commands

```bash
# SSH copy
scp -r user@host:/remote /local
scp -r /local user@host:/remote

# NFS mount
sudo mount -t nfs host:/share /mnt/nfs

# SMB mount
sudo mount -t cifs //host/share /mnt/smb -o guest

# Netcat transfer (receiver)
nc -l 1234 > file

# Netcat transfer (sender)
cat file | nc host 1234
```

---

## Quick Diagnostics

```bash
# Drive health
smartctl -H /dev/sda
smartctl -A /dev/sda | grep -iE "reallocated|pending|temperature"

# Bad blocks (read-only)
badblocks -nsv /dev/sda

# Performance test
hdparm -tT /dev/sda
dd if=/dev/sda of=/dev/null bs=1M count=1024 status=progress

# Temperature
sensors
smartctl -A /dev/sda | grep -i temp
```

---

## Quick Error Fixes

| Error | Command |
|-------|---------|
| "Structure needs cleaning" | `sudo ntfsfix /dev/sda2` |
| "Device is busy" | `sudo umount -l /dev/sda2` |
| "Wrong fs type" | `sudo mount -t ntfs-3g -o force /dev/sda2 /mnt/win` |
| "Read-only" | `sudo mount -o remount,rw /dev/sda2` |
| "No space" | `df -h` then compress: `dd \| gzip > file` |
| "I/O error" | Use `ddrescue` instead of `dd` |

---

## BlackArch Specific

```bash
# Update
sudo pacman -Syyu

# Install recovery tools
sudo pacman -S testdisk ddrescue ntfs-3g dislocker

# Install forensic tools
sudo pacman -S sleuthkit autopsy foremost

# Boot options (press Tab at boot)
blackarch.nox        # No graphics
blackarch.safe       # Safe mode
nomodeset           # Fix graphics
```

---

## One-Liners

```bash
# Image and hash in one command
dd if=/dev/sda bs=4M status=progress | tee >(sha256sum > hash.txt) > image.img

# Find and copy all documents
find /mnt/windows -name "*.doc*" -o -name "*.pdf" | xargs -I {} cp {} /mnt/backup/

# Backup user folders
for dir in Documents Downloads Desktop Pictures; do cp -r /mnt/win/Users/*/"$dir" /mnt/bkp/ 2>/dev/null; done

# Monitor dd progress
watch -n 5 'sudo kill -USR1 $(pgrep ^dd)'

# Check all partitions for errors
for p in /dev/sd*[0-9]; do echo "=== $p ==="; sudo fsck -n $p 2>&1 | head -5; done
```

---

## Emergency Numbers (Hardware)

| Symptom | Temperature | Action |
|---------|-------------|--------|
| HDD Clicking | N/A | STOP - Head crash |
| HDD Hot | >55°C | Cool before continuing |
| SSD Hot | >70°C | Cool before continuing |
| SMART "FAILING" | Any | Image immediately |
| I/O errors | Any | Use ddrescue, slow down |

---

## File Signatures for PhotoRec

```
# Common formats that recover well:
JPG:  FF D8 FF
PDF:  25 50 44 46
ZIP:  50 4B 03 04
PNG:  89 50 4E 47
MP4:  00 00 00 18 66 74 79 70
DOCX: 50 4B 03 04 (same as ZIP)
```

---

## Recovery Priority Order

1. **Image the drive first** (if hardware questionable)
2. **Check for encryption** (BitLocker key needed)
3. **Try simple mount** (ntfs-3g)
4. **Repair if needed** (ntfsfix, fsck)
5. **Use TestDisk** (partition/boot sector)
6. **File carving** (PhotoRec, Foremost - last resort)
