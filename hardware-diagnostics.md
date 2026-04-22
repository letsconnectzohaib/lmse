# Hardware Diagnostics Guide

Test and diagnose failing hardware before attempting recovery.

---

## Signs of Hardware Failure

### Hard Drive (HDD)

- Clicking, grinding, or beeping sounds
- Slow performance or freezing
- BIOS doesn't detect drive
- I/O errors in logs
- Bad sectors increasing

### SSD

- Suddenly disappears from BIOS
- Read-only mode (firmware lock)
- SMART errors
- Slow writes
- High temperature

---

## SMART Data Analysis

### Install smartmontools

```bash
sudo pacman -S smartmontools
```

### Check Drive Health

```bash
# Get all SMART data
sudo smartctl -a /dev/sda

# Quick health check
sudo smartctl -H /dev/sda

# Short test (2-5 minutes)
sudo smartctl -t short /dev/sda

# Long test (hours)
sudo smartctl -t long /dev/sda

# Check test results
sudo smartctl -l selftest /dev/sda
```

### Key SMART Attributes to Watch

```bash
# Parse important values
sudo smartctl -A /dev/sda | grep -E "(Reallocated|Pending|Uncorrectable|Temperature|Power_On)"
```

| Attribute | Critical Value | Meaning |
|-----------|---------------|---------|
| Reallocated_Sector_Ct | > 10 | Bad sectors remapped |
| Current_Pending_Sector | > 0 | Sectors waiting remap |
| Offline_Uncorrectable | > 0 | Unrecoverable errors |
| Temperature_Celsius | > 50°C | Overheating |
| Power_On_Hours | > 50000 | Very old drive |
| Wear_Leveling_Count (SSD) | < 10% remaining | SSD wearing out |

---

## Bad Block Scanning

### Non-Destructive Read Test

```bash
# Safe read-only test
sudo badblocks -nsv /dev/sda

# -n = non-destructive (read-only)
# -s = show progress
# -v = verbose
```

### Destructive Write Test (DESTROYS DATA)

```bash
# WARNING: This will erase all data!
sudo badblocks -wsv /dev/sda

# Only use if data already lost or drive being wiped
```

### Get Bad Block List

```bash
# Output bad blocks to file
sudo badblocks -v /dev/sda > /mnt/backup/badblocks.txt 2>&1

# Use with e2fsck to mark bad
sudo e2fsck -l /mnt/backup/badblocks.txt /dev/sda2
```

---

## Temperature Monitoring

### lm-sensors Setup

```bash
# Install
sudo pacman -S lm_sensors

# Detect sensors
sudo sensors-detect
# Answer YES to all prompts

# Check temperatures
sensors
```

### Continuous Monitoring

```bash
# Watch temperatures
watch -n 5 sensors

# Log to file
while true; do
  echo "$(date): $(sensors | grep -i 'core\|temp1' | head -5)" >> /tmp/temps.log
  sleep 60
done
```

### Drive Temperature Thresholds

| Component | Normal | Warning | Critical |
|-----------|--------|---------|----------|
| HDD | 30-40°C | 45-50°C | > 55°C |
| SSD | 30-45°C | 50-60°C | > 70°C |
| CPU | 40-60°C | 70-80°C | > 85°C |

---

## Disk Performance Testing

### hdparm - Basic Speed Test

```bash
# Cache read speed (not real)
sudo hdparm -tT /dev/sda

# Real read speed
sudo hdparm -t --direct /dev/sda
```

### dd - Sequential Speed Test

```bash
# Read speed test
sudo dd if=/dev/sda of=/dev/null bs=1M count=1024 status=progress

# Write speed test (DESTRUCTIVE)
sudo dd if=/dev/zero of=/dev/sda bs=1M count=1024 status=progress
```

### fio - Advanced Testing

```bash
# Install
sudo pacman -S fio

# Random read test (safer on partition, not whole disk)
sudo fio --name=randread --ioengine=libaio --iodepth=32 \
  --rw=randread --bs=4k --direct=1 --size=1G \
  --filename=/mnt/test/testfile --runtime=60
```

---

## RAM Testing

### MemTest86 (Most Reliable)

1. Download MemTest86 ISO
2. Create bootable USB
3. Boot from USB
4. Let it run multiple passes (hours)

### From Linux (Less Thorough)

```bash
# Install memtester
sudo pacman -S memtester

# Test 1GB of RAM
sudo memtester 1G 1

# Test all available RAM (careful!)
free -h  # Check available
sudo memtester 2G 3  # Test 2GB for 3 passes
```

---

## USB Drive Testing

### Identify Fake/Corrupted USB

```bash
# Check reported vs actual size
sudo pacman -S f3

# Write test data
f3write /mnt/usb

# Read and verify
f3read /mnt/usb

# Should report if sectors are bad
```

### USB Power Issues

```bash
# Check USB errors
dmesg | grep -iE "usb|over.current"

# Check USB device info
lsusb -v | less
```

---

## NVMe Specific Diagnostics

### nvme-cli

```bash
# Install
sudo pacman -S nvme-cli

# List NVMe devices
sudo nvme list

# SMART log
sudo nvme smart-log /dev/nvme0

# Error log
sudo nvme error-log /dev/nvme0

# Health check
sudo nvme smart-log /dev/nvme0 | grep -E "critical|percentage"
```

### NVMe Secure Erase

```bash
# Format NVMe (DESTRUCTIVE)
sudo nvme format /dev/nvme0 --ses=1

# Or use namespace format
sudo nvme format /dev/nvme0n1
```

---

## Cable and Connection Issues

### SATA Connection Problems

```bash
# Check for link errors
dmesg | grep -iE "ata|sata|link"

# Look for:
# "link down"
# "SATA link up 1.5 Gbps" (should be 3 or 6 Gbps)
# "device not ready"
```

### USB Connection Issues

```bash
# Check USB speed
lsusb -t

# Should show:
# 5000M for USB 3.0
# 480M for USB 2.0

# Check for errors
dmesg | grep -iE "usb|xhci|ehci" | tail -20
```

---

## Power Supply Issues

### Signs of Power Problems

- Drives disappearing/reappearing
- Random I/O errors
- Clicking sounds (head parking)
- Failed writes with no bad sectors

### Check Power (Software)

```bash
# Monitor for USB power issues
dmesg | grep -i "over-current"

# Check voltage (if sensors available)
sensors | grep -i "voltage\|Vcore"
```

---

## Firmware Issues

### SSD Firmware Problems

```bash
# Check SSD model
sudo hdparm -I /dev/sda | grep -i "model\|firmware"

# Check for firmware bugs online
# Some SSDs have known issues causing read-only mode
```

### HDD Firmware (Professional Only)

```bash
# Check for pending firmware issues
sudo smartctl -l scterc /dev/sda

# Some drives need TLER/ERC disabled for recovery
```

---

## Recovery from Failing Drives

### Immediate Actions

```bash
# 1. STOP using the drive immediately
# 2. Check temperature - cool it down if hot

# 3. Check SMART
echo "SMART status:"
sudo smartctl -H /dev/sda

# 4. If failing, image it NOW with ddrescue
sudo ddrescue -n /dev/sda /mnt/backup/failing.img /mnt/backup/failing.log

# 5. Work only on the image, not original
```

### Cooling Techniques

```bash
# Monitor temperature during recovery
while true; do
  sudo hddtemp /dev/sda 2>/dev/null || sudo smartctl -A /dev/sda | grep Temperature
  sleep 60
done

# If overheating, pause recovery
echo "Overheating! Pausing..."
sleep 300  # Wait 5 minutes
```

### Intermittent Drive Handling

```bash
# For drives that disappear/reappear
# Use ddrescue with frequent sync

sudo ddrescue --no-scrape --no-trim --max-retries=1 \
  /dev/sda /mnt/backup/intermittent.img /mnt/backup/intermittent.log

# Retry bad sectors separately
sudo ddrescue -r3 -d /dev/sda /mnt/backup/intermittent.img /mnt/backup/intermittent.log
```

---

## Error Interpretation

### Kernel Messages (dmesg)

```bash
# Check disk errors
sudo dmesg | grep -iE "ata|scsi|i/o error|failed|corrupt"

# Common patterns:
# "I/O error" = Hardware or filesystem issue
# "Medium Error" = Bad sector
# "Not Ready" = Drive spinning up or failing
# "ABRT" = Command aborted, firmware issue
```

### Interpreting SMART Errors

```bash
# Check error log
sudo smartctl -l error /dev/sda

# Recent errors with timestamps
sudo smartctl -l xerror /dev/sda 2>/dev/null
```

---

## When to Stop Recovery Attempts

### Critical Warning Signs

| Sign | Action |
|------|--------|
| Clicking sounds | Stop immediately - head crash |
| Temperature > 60°C | Cool down before continuing |
| SMART shows "FAILING_NOW" | Image what you can quickly |
| Drive disappears during scan | Hardware failure, use professional |
| Burning smell | STOP - electrical damage |
| Smoke | STOP - unplug immediately |

---

## Quick Commands Reference

```bash
# Full hardware check
sudo smartctl -a /dev/sda | grep -E "(Health|Reallocated|Pending|Temperature|result)"

# Temperature only
sudo smartctl -A /dev/sda | grep -i temp

# Quick bad block check (read-only)
sudo badblocks -nsv /dev/sda

# Performance test
sudo hdparm -tT /dev/sda

# Check for recent errors
sudo dmesg | grep -iE "error|fail|corrupt" | tail -20
```
