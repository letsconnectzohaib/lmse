# Forensic Analysis Tools

Advanced tools for forensic analysis and evidence preservation.

---

## When to Use Forensic Tools

- Legal evidence preservation
- Investigation of security incidents
- Malware analysis
- Data theft investigation
- Court-admissible documentation

---

## The Sleuth Kit (TSK)

### Install

```bash
sudo pacman -S sleuthkit
```

### Filesystem Analysis

```bash
# Identify filesystem type
fsstat /dev/sda2

# List directory contents
fls -r /dev/sda2 /

# List deleted files
fls -r -p /dev/sda2 /

# Get file details
istat /dev/sda2 1234  # inode number from fls

# Extract file
icat /dev/sda2 1234 > recovered-file.doc
```

### Timeline Analysis

```bash
# Create timeline
mactime -b /dev/sda2 -d -y > timeline.txt

# Bodyfile creation
fls -r -m / /dev/sda2 > bodyfile.txt
mactime -b bodyfile.txt -d -y > timeline.csv
```

---

## Autopsy (GUI Forensic Browser)

### Install and Run

```bash
# Install
sudo pacman -S autopsy

# Run
sudo autopsy

# Access via browser: http://localhost:9999/autopsy
```

### Autopsy Workflow

1. **Create Case**
   - Case name: "Investigation_2024"
   - Base directory: `/mnt/cases/`

2. **Add Host**
   - Host name: "SuspectPC"

3. **Add Image**
   - Select disk image
   - Import or add existing

4. **Analyze**
   - File browsing
   - Keyword search
   - Timeline
   - Hash analysis
   - Deleted file recovery

---

## Hashing for Evidence Integrity

### Calculate Hashes

```bash
# MD5 (fast, less secure)
md5sum /mnt/backup/evidence.img

# SHA256 (recommended)
sha256sum /mnt/backup/evidence.img

# SHA1 (legacy)
sha1sum /mnt/backup/evidence.img

# Multiple algorithms at once
rhash --md5 --sha256 --sha1 evidence.img
```

### Verify Image Integrity

```bash
# Create hash when imaging
dd if=/dev/sda bs=4M status=progress | tee >(sha256sum > /mnt/backup/sda.sha256) > /mnt/backup/sda.img

# Verify later
sha256sum -c /mnt/backup/sda.sha256
```

### Bulk File Hashing

```bash
# Hash all files in directory
find /mnt/evidence -type f -exec sha256sum {} + > hashes.txt

# Compare against known good/bad lists
while read hash file; do
  if grep -q "$hash" known-malware-hashes.txt; then
    echo "MALWARE: $file"
  fi
done < hashes.txt
```

---

## Evidence Imaging Standards

### DCFLDD (Forensic DD)

```bash
# Install
sudo pacman -S dcfldd

# Forensic imaging with verification
sudo dcfldd if=/dev/sda of=/mnt/evidence/suspect.img \
  bs=4M hash=sha256 hashlog=/mnt/evidence/suspect.sha256 \
  status=on

# With progress and multiple hashes
sudo dcfldd if=/dev/sda of=/mnt/evidence/suspect.img \
  bs=4M hash=md5,sha256 \
  md5log=/mnt/evidence/suspect.md5 \
  sha256log=/mnt/evidence/suspect.sha256 \
  status=on
```

### Guymager (GUI Forensic Imager)

```bash
# Install
sudo pacman -S guymager

# Run
sudo guymager

# Features:
# - E01 (Expert Witness Format)
# - Raw (dd) format
# - Compression
# - Automatic hashing
# - Resume capability
```

### E01 Format (Industry Standard)

```bash
# Create E01 with ewf-tools
sudo pacman -S libewf

# Acquire image
sudo ewfacquire /dev/sda

# Follow prompts:
# - Case number
# - Evidence number
# - Examiner name
# - Compression level
# - Split size

# Verify E01
sudo ewfverify /mnt/evidence/suspect.E01
```

---

## File Carving

### Foremost

```bash
# Install
sudo pacman -S foremost

# Basic carving
sudo foremost -i /dev/sda2 -o /mnt/carved/

# Specific file types only
sudo foremost -i /dev/sda2 -t jpg,pdf,doc,xls -o /mnt/carved/

# From image file
sudo foremost -i disk.img -o /mnt/carved/

# Quiet mode (no screen output)
sudo foremost -Q -i /dev/sda2 -o /mnt/carved/
```

### Scalpel

```bash
# Install
sudo pacman -S scalpel

# Edit config to add file types
sudo nano /etc/scalpel/scalpel.conf
# Uncomment lines for file types you want

# Run carving
sudo scalpel /dev/sda2 -o /mnt/carved/

# More aggressive (slower)
sudo scalpel -b 512 /dev/sda2 -o /mnt/carved/
```

### Bulk Extractor

```bash
# Install
sudo pacman -S bulk-extractor

# Extract all artifacts
bulk_extractor -o /mnt/bulk-out/ /mnt/evidence/disk.img

# Features:
# - Email addresses
# - Credit card numbers
# - URLs
# - Phone numbers
# - EXIF data
# - Social Security Numbers
# - Keyword search

# View results
ls /mnt/bulk-out/
cat /mnt/bulk-out/email.txt
```

---

## Memory Analysis

### Create Memory Dump

```bash
# From running system
sudo pacman -S lime-dkms

# Load module
sudo insmod /lib/modules/$(uname -r)/kernel/drivers/lime.ko \
  "path=/mnt/evidence/memory.lime format=lime"

# Or use /proc/kcore (limited)
sudo dd if=/proc/kcore of=/mnt/evidence/memory.raw bs=1M
```

### Volatility Framework

```bash
# Install volatility3
sudo pacman -S volatility3

# Identify profile
vol.py -f memory.lime windows.info

# List processes
vol.py -f memory.lime windows.pslist

# Network connections
vol.py -f memory.lime windows.netstat

# File handles
vol.py -f memory.lime windows.handles

# Command history
vol.py -f memory.lime windows.cmdscan
```

---

## String Extraction

### Basic Strings

```bash
# Extract all readable strings
sudo strings /dev/sda2 > /mnt/evidence/strings.txt

# Unicode strings (Windows)
sudo strings -el /dev/sda2 > /mnt/evidence/unicode-strings.txt

# Minimum length (reduce noise)
sudo strings -n 10 /dev/sda2 > /mnt/evidence/long-strings.txt
```

### Contextual Strings (Binwalk)

```bash
# Install
sudo pacman -S binwalk

# Extract and analyze
binwalk -e /dev/sda2

# Just identify
binwalk /dev/sda2

# Extract specific offset
binwalk -M -e /mnt/evidence/disk.img
```

---

## Log Analysis

### Windows Event Logs

```bash
# Install parsing tools
sudo pacman -S python-evtx

# Convert to XML
evtx_dump.py /mnt/windows/Windows/System32/winevt/Logs/Security.evtx > security.xml

# Or use grep on raw
grep -a "EventID" /mnt/windows/Windows/System32/winevt/Logs/Security.evtx | head -20
```

### Registry Analysis

```bash
# Install regripper
sudo pacman -S regripper

# Extract Windows registry data
rip.pl -r /mnt/windows/Windows/System32/config/SAM -f sam
rip.pl -r /mnt/windows/Windows/System32/config/SOFTWARE -f software
rip.pl -r /mnt/windows/Windows/System32/config/SYSTEM -f system

# Recent docs, USB history, user activity
rip.pl -r /mnt/windows/Windows/System32/config/SOFTWARE -p usbdevices
```

---

## Chain of Custody Documentation

### Metadata Recording

```bash
#!/bin/bash
# save as /usr/local/bin/log-evidence.sh

EVIDENCE="$1"
CASE="$2"
LOG="/mnt/cases/$CASE/evidence.log"

mkdir -p "/mnt/cases/$CASE"

echo "=== Evidence Log ===" >> "$LOG"
echo "Date: $(date)" >> "$LOG"
echo "Examiner: $(whoami)" >> "$LOG"
echo "Evidence: $EVIDENCE" >> "$LOG"
echo "Case: $CASE" >> "$LOG"
echo "" >> "$LOG"

# Hardware info
echo "Drive Info:" >> "$LOG"
sudo smartctl -i "$EVIDENCE" >> "$LOG" 2>/dev/null
echo "" >> "$LOG"

# Hashes
echo "SHA256:" >> "$LOG"
sha256sum "$EVIDENCE" >> "$LOG"
echo "" >> "$LOG"

echo "Log entry complete" >> "$LOG"
```

---

## Anti-Forensics Detection

### Check for Wiping Tools

```bash
# Search for wiping tool artifacts
grep -a "shred\|wipe\|eraser\|ccleaner\|cipher" /dev/sda2 | head -20

# Check for wiped areas (patterns)
sudo xxd /dev/sda2 | grep -E "(0000 0000 0000|ffff ffff ffff)" | head -10
```

### Timeline Anomalies

```bash
# Find suspicious time gaps
mactime -b bodyfile.txt | awk '{print $1}' | uniq -c | sort -rn

# Files with future dates
find /mnt/evidence -type f -newermt "2025-01-01" -ls
```

---

## Quick Forensic Commands

```bash
# Fast evidence preservation
sudo dcfldd if=/dev/sda of=/mnt/evidence/evidence.img hash=sha256

# Extract deleted files
sudo photorec /dev/sda2

# Timeline creation
fls -r -m / /dev/sda2 | mactime -b - -d -y > timeline.csv

# String search for keywords
sudo strings /dev/sda2 | grep -i "password\|secret\|confidential"

# Hash all files
find /mnt/evidence -type f -exec sha256sum {} \; > hashes.sha256
```
