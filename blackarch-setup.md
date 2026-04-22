# BlackArch Linux Recovery Setup

Complete guide for setting up BlackArch Linux for data recovery operations.

---

## Creating BlackArch Live USB

### Option 1: Official ISO

```bash
# Download latest BlackArch ISO
wget https://blackarch.org/iso/blackarch-linux-full.iso

# Write to USB (replace /dev/sdX with your USB device)
sudo dd if=blackarch-linux-full.iso of=/dev/sdX bs=4M status=progress
sync
```

### Option 2: Persistent USB (Recommended)

```bash
# Create partitions: 4GB for ISO, rest for persistence
sudo fdisk /dev/sdX
# n -> p -> 1 -> [enter] -> +4G
# n -> p -> 2 -> [enter] -> [enter]
# w

# Write ISO to first partition
dd if=blackarch-linux-full.iso of=/dev/sdX1 bs=4M

# Format persistence partition
mkfs.ext4 -L persistence /dev/sdX2

# Create persistence.conf
mkdir -p /mnt/persist
mount /dev/sdX2 /mnt/persist
echo "/home" > /mnt/persist/persistence.conf
echo "/var/cache/pacman/pkg" >> /mnt/persist/persistence.conf
umount /mnt/persist
```

---

## Essential BlackArch Recovery Tools

```bash
# Update system first
sudo pacman -Syyu

# Install core recovery tools
sudo pacman -S testdisk ddrescue ntfs-3g dislocker

# Install forensic tools
sudo pacman -S sleuthkit autopsy foremost scalpel

# Install disk utilities
sudo pacman -S gparted parted fdisk gdisk smartmontools

# Install network recovery tools
sudo pacman -S rsync openssh nfs-utils samba

# Install analysis tools
sudo pacman -S hexedit xxd binwalk
```

---

## BlackArch Recovery Categories

```bash
# List all forensics tools
pacman -Sg blackarch-forensic

# List all recovery tools
pacman -Sg blackarch-defensive

# Install all forensics tools (huge download)
sudo pacman -S blackarch-forensic
```

---

## Setting Up Recovery Environment

### Mount Points Structure

```bash
# Create standard mount points
sudo mkdir -p /mnt/{source,backup,recovery,image,temp}

# Set permissions
sudo chmod 777 /mnt/{source,backup,recovery,image,temp}
```

### Configure Pacman for Offline Use

```bash
# Download packages to USB for offline install
sudo pacman -Sw testdisk ddrescue ntfs-3g dislocker

# Packages stored in /var/cache/pacman/pkg/
# Copy to USB:
sudo cp /var/cache/pacman/pkg/*.pkg.tar.zst /mnt/usb/packages/

# Install from USB on offline system:
sudo pacman -U /mnt/usb/packages/*.pkg.tar.zst
```

---

## Network Boot Setup (PXE)

For recovering multiple machines:

```bash
# Install PXE server tools
sudo pacman -S dnsmasq syslinux

# Configure dnsmasq for PXE boot
sudo tee /etc/dnsmasq.conf << 'EOF'
interface=eth0
dhcp-range=192.168.1.100,192.168.1.200,12h
dhcp-boot=pxelinux.0,pxeserver,192.168.1.1
enable-tftp
tftp-root=/srv/tftp
EOF

# Copy BlackArch PXE files
sudo mkdir -p /srv/tftp
sudo cp /usr/lib/syslinux/bios/pxelinux.0 /srv/tftp/
sudo cp -r /boot/blackarch /srv/tftp/

# Start server
sudo systemctl start dnsmasq
```

---

## Troubleshooting BlackArch Boot

### Issue: Black screen after boot

```bash
# Try safe graphics mode at boot menu
# Press Tab and add to kernel line:
blackarch.nox blackarch.safe

# Or for specific graphics:
nomodeset
intel.modeset=0
nouveau.modeset=0
```

### Issue: Can't find drives

```bash
# Load all disk modules
sudo modprobe usb_storage
sudo modprobe sd_mod
sudo modprobe nvme
sudo modprobe ahci

# Rescan SCSI bus
echo "- - -" > /sys/class/scsi_host/host0/scan
```

### Issue: Keyboard layout wrong

```bash
# Set US layout
loadkeys us

# Set your layout
loadkeys de  # German
loadkeys fr  # French
```

---

## BlackArch Window Manager for Recovery

### OpenBox (Lightweight)

```bash
# Start graphical environment
startx

# Launch file manager
spacefm /mnt/source &

# Launch terminal
termite &
```

### XFCE (More familiar)

```bash
# Install XFCE if needed
sudo pacman -S xfce4 xfce4-goodies

# Start XFCE
startxfce4
```

---

## Custom Recovery Scripts

### Auto-Mount Script

```bash
#!/bin/bash
# save as /usr/local/bin/auto-mount.sh

for dev in /dev/sd*; do
    if [[ "$dev" =~ ^/dev/sd[a-z]$ ]]; then
        continue
    fi
    
    label=$(lsblk -no LABEL "$dev" 2>/dev/null)
    uuid=$(lsblk -no UUID "$dev" 2>/dev/null)
    
    mkdir -p "/mnt/$uuid"
    mount "$dev" "/mnt/$uuid" 2>/dev/null && echo "Mounted $dev to /mnt/$uuid"
done
```

### Quick Backup Script

```bash
#!/bin/bash
# save as /usr/local/bin/quick-backup.sh

SOURCE="${1:-/mnt/source}"
DEST="${2:-/mnt/backup}"
USER="${3:-*}"

echo "Backing up from $SOURCE/Users/$USER"
echo "Destination: $DEST"

rsync -avh --progress "$SOURCE/Users/$USER/Documents" "$DEST/"
rsync -avh --progress "$SOURCE/Users/$USER/Downloads" "$DEST/"
rsync -avh --progress "$SOURCE/Users/$USER/Desktop" "$DEST/"
rsync -avh --progress "$SOURCE/Users/$USER/Pictures" "$DEST/"

echo "Backup complete!"
```

---

## Hardware Diagnostics on BlackArch

### Check Drive Health

```bash
# Install smartmontools if needed
sudo pacman -S smartmontools

# Check SMART data
sudo smartctl -a /dev/sda

# Run short test
sudo smartctl -t short /dev/sda
sudo smartctl -l selftest /dev/sda

# Run long test (takes hours)
sudo smartctl -t long /dev/sda
```

### Temperature Monitoring

```bash
# Install sensors
sudo pacman -S lm_sensors
sudo sensors-detect
sensors
```

### Bad Block Scan

```bash
# Non-destructive bad block check
sudo badblocks -nsv /dev/sda

# Read-only check (safe)
sudo badblocks -sv /dev/sda
```

---

## Emergency Shell Commands

```bash
# When GUI fails, use these in TTY (Ctrl+Alt+F2):

# List all block devices
lsblk -f

# Check kernel messages
dmesg | tail -50

# Check mounted filesystems
cat /proc/mounts

# Force mount NTFS
sudo mount -t ntfs-3g -o force,rw,remove_hiberfile /dev/sda2 /mnt/source

# Check for hardware errors
dmesg | grep -iE "error|fail|corrupt"
```

---

## BlackArch to USB Persistence

Save your recovery work across reboots:

```bash
# Create persistence partition (at least 8GB)
sudo fdisk /dev/sdX
# Create ext4 partition

# Mount and setup
sudo mkdir -p /mnt/persist
sudo mount /dev/sdX2 /mnt/persist

# Enable full persistence
echo "/ union" | sudo tee /mnt/persist/persistence.conf

# Or selective persistence:
cat << 'EOF' | sudo tee /mnt/persist/persistence.conf
/home
/root
/var/cache/pacman/pkg
/etc
/usr/local/bin
EOF

sudo umount /mnt/persist
```

---

## Quick Reference: BlackArch Recovery Boot Modes

| Boot Option | Use When |
|-------------|----------|
| `BlackArch Linux` | Normal boot |
| `BlackArch Linux (nomodeset)` | Graphics issues |
| `BlackArch Linux (RAM)` | Run entirely from RAM |
| `BlackArch Linux (safe)` | Maximum compatibility |
| `MemTest86` | Test RAM |
