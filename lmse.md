# Data Recovery Guides

Complete guides for recovering data from corrupted, encrypted, or unbootable Windows systems.

---

## Quick Navigation

| Guide | When to Use |
|-------|-------------|
| [Linux Data Recovery](linux-data-recovery.md) | Boot from Linux Live USB, mount drives, copy files |
| [Windows Recovery](windows-recovery.md) | Use Windows Recovery USB for boot repair |
| [BitLocker Guide](bitlocker-guide.md) | Handle BitLocker encrypted drives |
| [No-Key Recovery](no-key-recovery.md) | Last resort when no keys/passwords available |

---

## Quick Diagnosis

### From Linux Live USB:

```bash
# Check disks
lsblk -f
```

| FSTYPE | Meaning | Next Step |
|--------|---------|-----------|
| `ntfs` | Normal Windows | [Linux Guide](linux-data-recovery.md) |
| `unknown` / blank | Damaged or BitLocker | [BitLocker Guide](bitlocker-guide.md) |
| `BitLocker` | Encrypted | [BitLocker Guide](bitlocker-guide.md) |
| `crypto_LUKS` | Linux encryption | Password required |

---

## Common Scenarios

### Scenario 1: Windows Won't Boot, No Encryption

1. Boot Linux Live USB
2. Mount Windows partition
3. Copy files to external USB
4. → [Linux Data Recovery Guide](linux-data-recovery.md)

### Scenario 2: BitLocker Encrypted Drive

1. Check [BitLocker Guide](bitlocker-guide.md)
2. Find recovery key in Microsoft account
3. Use `dislocker` from Linux or `manage-bde` from Windows Recovery

### Scenario 3: No Recovery Key Available

1. Try PhotoRec for raw file recovery
2. Check Microsoft account one more time
3. Consider professional recovery if data is critical
4. → [No-Key Recovery Guide](no-key-recovery.md)

### Scenario 4: Boot Issues Only

1. Windows Recovery USB
2. Run `bootrec` commands
3. Run `chkdsk`
4. → [Windows Recovery Guide](windows-recovery.md)

---

## Emergency Quick Commands

### Linux:

```bash
# List drives
lsblk -f

# Mount Windows drive
sudo mount /dev/sda2 /mnt/windows

# Copy files
sudo cp -r /mnt/windows/Users/Name/Documents /mnt/usb/
```

### Windows Recovery:

```cmd
# Check BitLocker
manage-bde -status

# Fix boot
bootrec /fixmbr
bootrec /rebuildbcd

# Disk check
chkdsk C: /f
```

---

## Tools Overview

| Tool | Platform | Purpose |
|------|----------|---------|
| `dislocker` | Linux | Mount BitLocker drives |
| `ntfsfix` | Linux | Repair NTFS errors |
| `photorec` | Linux | Raw file recovery |
| `bootrec` | Windows | Boot repair |
| `manage-bde` | Windows | BitLocker control |
| `chntpw` | Linux | Windows password reset |

---

## Important Notes

- **BitLocker without key = unrecoverable** (by design)
- Always work on disk images, not original drives
- Check Microsoft account for recovery keys: https://account.microsoft.com/devices/recoverykey
- PhotoRec recovers files but loses folder structure and filenames
