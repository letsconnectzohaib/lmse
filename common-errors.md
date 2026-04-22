# Common Errors and Fixes

Troubleshoot common recovery errors with specific solutions.

---

## Linux Mount Errors

### "Structure needs cleaning"

**Cause:** Corrupted filesystem

**Fix:**
```bash
# For NTFS
sudo ntfsfix /dev/sda2

# For ext4
sudo fsck -y /dev/sda2

# For FAT32
sudo fsck.vfat -a /dev/sda1
```

---

### "Device is busy"

**Cause:** Partition mounted or in use

**Fix:**
```bash
# Find what's using it
sudo lsof /dev/sda2
sudo fuser -m /dev/sda2

# Force unmount
sudo umount -l /dev/sda2
# or
sudo umount -f /dev/sda2

# Lazy unmount (cleanup later)
sudo umount -l /mnt/point
```

---

### "Wrong fs type, bad option, bad superblock"

**Cause:** Filesystem not recognized

**Fix:**
```bash
# Check actual filesystem
sudo file -s /dev/sda2

# If NTFS, force mount
sudo mount -t ntfs-3g -o force,rw /dev/sda2 /mnt/windows

# If ext4 with bad superblock, use backup
sudo e2fsck -b 32768 /dev/sda2

# If unknown, might be BitLocker - check signature
sudo xxd -l 16 /dev/sda2
# Look for "-FVE-FS-" (BitLocker) or "NTFS" 
```

---

### "NTFS is inconsistent"

**Cause:** Windows fast boot or hibernation

**Fix:**
```bash
# Clear hibernation file
sudo ntfsfix -b /dev/sda2

# Or mount with remove_hiberfile
sudo mount -t ntfs-3g -o remove_hiberfile,rw /dev/sda2 /mnt/windows

# If still fails, from Windows Recovery:
chkdsk C: /f
```

---

### "Read-only file system"

**Cause:** Filesystem mounted ro due to errors

**Fix:**
```bash
# Check why it's ro
mount | grep sda2

# Remount as rw
sudo mount -o remount,rw /dev/sda2

# If NTFS, force it
sudo mount -t ntfs-3g -o force,rw /dev/sda2 /mnt/windows
```

---

## BitLocker / Encryption Errors

### "dislocker: ERROR: Cannot parse volume header"

**Cause:** Not BitLocker, or severely damaged

**Fix:**
```bash
# Verify it's BitLocker
sudo xxd -l 16 /dev/sda2 | head -1

# If not "-FVE-FS-", it's not BitLocker
# Try normal mount
sudo mount /dev/sda2 /mnt/test

# If BitLocker confirmed but fails, try recovery key instead of password
sudo dislocker /dev/sda2 -pYOUR-48-DIGIT-KEY -- /mnt/dislocker
```

---

### "dislocker: Authentication failed"

**Cause:** Wrong password or recovery key

**Fix:**
```bash
# Verify key format (no spaces, 48 digits)
echo "YOUR-KEY" | tr -d '-' | wc -c  # Should be 48

# Try with dashes
sudo dislocker /dev/sda2 -pXXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX-XXXXXX -- /mnt/dislocker

# Check Microsoft account for correct key
# https://account.microsoft.com/devices/recoverykey
```

---

### "The disk contains an unclean file system"

**Cause:** BitLocker metadata inconsistent

**Fix:**
```bash
# Force metadata repair
sudo dislocker-fuse -v -pYOUR-KEY /dev/sda2 /mnt/dislocker

# Or try without metadata validation
sudo dislocker -V /dev/sda2 -pYOUR-KEY -- /mnt/dislocker
```

---

## DD / DDRescue Errors

### "No space left on device"

**Cause:** Destination full

**Fix:**
```bash
# Check space
df -h /mnt/backup

# Compress on the fly
sudo dd if=/dev/sda bs=4M | gzip > /mnt/backup/image.img.gz

# Or use split
sudo dd if=/dev/sda bs=4M | split -b 4G - /mnt/backup/image.img.part

# Use external drive with more space
sudo dd if=/dev/sda of=/mnt/external/image.img bs=4M status=progress
```

---

### "Input/output error" during dd

**Cause:** Bad sectors or failing drive

**Fix:**
```bash
# Switch to ddrescue immediately!
sudo ddrescue -n /dev/sda /mnt/backup/image.img /mnt/backup/logfile

# If ddrescue also fails, try direct IO
sudo ddrescue -d /dev/sda /mnt/backup/image.img /mnt/backup/logfile
```

---

### "ddrescue: Logfile exists"

**Cause:** Previous incomplete run

**Fix:**
```bash
# Resume from where it stopped
sudo ddrescue -r3 /dev/sda /mnt/backup/image.img /mnt/backup/existing.log

# Or view current status
sudo ddrescue --status /mnt/backup/existing.log
```

---

### "dd: failed to open '/dev/sda': Permission denied"

**Cause:** Missing sudo or device busy

**Fix:**
```bash
# Use sudo
sudo dd if=/dev/sda of=/mnt/backup/image.img bs=4M

# Check if device in use
sudo lsof /dev/sda
sudo fuser /dev/sda
```

---

## SSH / Network Errors

### "Connection refused"

**Cause:** SSH not running or firewall

**Fix:**
```bash
# On destination machine
sudo systemctl start sshd
sudo systemctl enable sshd

# Check if running
sudo ss -tlnp | grep 22

# Check firewall
sudo iptables -L | grep 22
```

---

### "Permission denied (publickey)"

**Cause:** Password auth disabled

**Fix:**
```bash
# Force password auth
scp -o PreferredAuthentications=password -o PubkeyAuthentication=no file user@host:/dest/

# Or enable password auth on server
# Edit /etc/ssh/sshd_config:
# PasswordAuthentication yes
sudo systemctl restart sshd
```

---

### "rsync: connection unexpectedly closed"

**Cause:** Network timeout or large file issues

**Fix:**
```bash
# Resume with partial files
rsync -avzP --partial /source/ user@host:/dest/

# Increase timeout
rsync -avz --timeout=300 /source/ user@host:/dest/

# Exclude large files initially
rsync -avz --max-size=100M /source/ user@host:/dest/
```

---

## TestDisk / PhotoRec Errors

### "No partition found or selected"

**Cause:** Wrong partition table type

**Fix:**
```bash
# In TestDisk:
# 1. Check if drive is GPT or MBR
sudo fdisk -l /dev/sda | grep -i "disklabel\|partition table"

# 2. Select correct type in TestDisk:
# [Intel] for MBR
# [EFI GPT] for GPT

# 3. Run Deeper Search if Quick Search fails
```

---

### "List files won't work"

**Cause:** Filesystem too damaged

**Fix:**
```bash
# Try different approach:
# 1. Use ddrescue to image first
sudo ddrescue /dev/sda2 /mnt/backup/partition.img /mnt/backup/part.log

# 2. Mount the image
sudo losetup -fP /mnt/backup/partition.img

# 3. Run TestDisk on image
sudo testdisk /dev/loop0

# 4. Or use PhotoRec directly
sudo photorec /dev/loop0
```

---

### "PhotoRec: No file found"

**Cause:** Wrong file selection or all data overwritten

**Fix:**
```bash
# In PhotoRec:
# 1. Press 'f' to modify file options
# 2. Enable more file types (not just defaults)
# 3. For custom files, add signatures:
# Edit /etc/photorec.sig

# If still nothing, data may be overwritten
# Try carving with foremost instead:
sudo foremost -i /dev/sda2 -o /mnt/recovery/
```

---

## Boot Errors

### "Error: no such device"

**Cause:** GRUB can't find partition

**Fix:**
```bash
# From Live USB, chroot into system
sudo mount /dev/sda2 /mnt
sudo mount /dev/sda1 /mnt/boot/efi  # if UEFI
sudo mount --bind /dev /mnt/dev
sudo mount --bind /proc /mnt/proc
sudo mount --bind /sys /mnt/sys
sudo chroot /mnt

# Reinstall GRUB
grub-install /dev/sda
update-grub

exit
sudo umount -R /mnt
```

---

### "BOOTMGR is missing"

**Cause:** Windows bootloader damaged

**Fix:**
```bash
# From Windows Recovery USB:
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd

# If UEFI:
bcdboot C:\Windows /f UEFI
```

---

## Hardware Detection Errors

### "lsblk shows no partitions"

**Cause:** Partition table corrupt or driver issue

**Fix:**
```bash
# Force kernel to rescan
sudo partprobe /dev/sda

# Or manually rescan
echo 1 | sudo tee /sys/block/sda/device/rescan

# Check if disk detected at all
sudo fdisk -l /dev/sda

# If no partitions found, use TestDisk
sudo testdisk /dev/sda
# > [Analyse] > [Deeper Search]
```

---

### "USB device not recognized"

**Cause:** Power issue or bad USB port

**Fix:**
```bash
# Check kernel messages
dmesg | tail -30

# Look for:
# - "over-current"
# - "device descriptor read/64, error"

# Try different USB port (use USB 2.0 for power-hungry drives)

# Check USB power limits
lsusb -v 2>/dev/null | grep -i "MaxPower"
```

---

## Package Installation Errors

### "target not found: package"

**Cause:** Package not in repos or typo

**Fix:**
```bash
# Update database
sudo pacman -Syy

# Search for correct name
pacman -Ss testdisk

# For BlackArch, check groups
pacman -Sg | grep -i blackarch

# Install from BlackArch repo
sudo pacman -S blackarch/foremost
```

---

### "failed to commit transaction (conflicting files)"

**Cause:** File ownership conflicts

**Fix:**
```bash
# Overwrite with package version
sudo pacman -S package --overwrite '*'

# Or specific file
sudo pacman -S package --overwrite /path/to/file
```

---

## Quick Error Diagnosis

```bash
# General system health check
dmesg | grep -iE "error|fail|corrupt|warn" | tail -20

# Check disk errors specifically
sudo smartctl -H /dev/sda
sudo smartctl -l error /dev/sda

# Check filesystem errors
sudo fsck -n /dev/sda2  # dry run

# Check memory errors
dmesg | grep -i "memory\|oom\|segfault"
```

---

## Emergency Recovery Mode

When everything fails:

```bash
# 1. Boot BlackArch with minimal options
# At boot menu press Tab, add: blackarch.nox nomodeset

# 2. Drop to emergency shell if boot fails
# Add to kernel line: systemd.unit=emergency.target

# 3. Mount root manually
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot

# 4. Basic tools available in initramfs
/bin/dd if=/dev/sda of=/mnt/backup bs=1M count=100
```
