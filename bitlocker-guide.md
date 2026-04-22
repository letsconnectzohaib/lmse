# BitLocker Guide

Everything about handling BitLocker encrypted drives.

---

## What is BitLocker

- Full disk encryption built into Windows Pro/Enterprise
- Automatically enabled on many modern PCs with TPM
- **AES encryption** - mathematically impossible to crack without key

---

## Check if Drive is BitLocker Encrypted

### From Linux:

```bash
# Check filesystem type
lsblk -f

# Confirm BitLocker signature
sudo xxd -l 256 /dev/sda2 | head -5
# Look for "-FVE-FS-" at start
```

### From Windows Recovery:

```cmd
manage-bde -status
```

---

## Finding Your Recovery Key

### 1. Microsoft Account (Most Common)

Visit: **https://account.microsoft.com/devices/recoverykey**

- Sign in with same account used on PC
- Look for device name matching your PC
- Copy the 48-digit recovery key

### 2. Active Directory / Azure AD

- Check with your IT administrator
- BitLocker keys stored in AD for business PCs

### 3. Physical Printout

- Check documentation that came with PC
- Some manufacturers print key on card

### 4. USB Drive

- Some users save `BitLocker Recovery Key.txt` to USB

---

## Unlock BitLocker from Linux

### Using dislocker:

```bash
# Install dislocker
sudo pacman -S dislocker  # Arch/BlackArch
sudo apt install dislocker # Ubuntu/Debian

# Create mount points
sudo mkdir -p /mnt/dislocker /mnt/windows

# Unlock with recovery key
sudo dislocker /dev/sda2 -pYOUR-48-DIGIT-KEY -- /mnt/dislocker

# Or unlock with password (if set)
sudo dislocker /dev/sda2 -u -- /mnt/dislocker

# Mount decrypted drive
sudo mount /mnt/dislocker/dislocker-file /mnt/windows

# Access files
ls /mnt/windows/Users/YourName/Documents
```

---

## No Recovery Key - What Now?

### Reality Check:
- **No tool exists** to crack BitLocker
- **No Linux tool** can bypass it
- **Professional services** can only try chip-off forensics (expensive)

### Partial Recovery Options:

#### PhotoRec - File Carving

Ignores encryption headers, scans for file signatures:

```bash
sudo pacman -S testdisk
sudo photorec /dev/sda2
```

**Limitations:**
- No folder structure
- No filenames (f0000001.jpg, etc.)
- Only recoverable files by signature (photos, docs, videos)
- Partial files likely

---

## Disable BitLocker (When You Have Access)

### From Windows:

```cmd
# Open CMD as Admin
manage-bde -off C:
```

Or via GUI:
1. Open **BitLocker Drive Encryption** in Control Panel
2. Click **Turn off BitLocker**
3. Wait for decryption (hours for large drives)

---

## Prevent Future Lockouts

1. **Save recovery key to Microsoft account**
2. **Print key and store safely**
3. **Backup to USB/external drive**
4. **Disable auto-encryption** if not needed:

```cmd
# Disable automatic device encryption
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\EnhancedStorageDevices" /v TCGSecurityActivationDisabled /t REG_DWORD /d 1 /f
```

---

## Troubleshooting dislocker

### Error: "Unable to find a valid metadata header"

```bash
# Force metadata search
sudo dislocker-fuse -v -pYOUR-KEY /dev/sda2 /mnt/dislocker
```

### Error: "Wrong password or recovery key"

- Double-check key (no spaces, all 48 digits)
- Try different partition
- Confirm it's BitLocker not other encryption

---

## Quick Commands

```bash
# Check all partitions
for part in /dev/sda*; do echo "=== $part ===" && sudo file -s "$part" 2>/dev/null; done

# Test dislocker without mounting
sudo dislocker /dev/sda2 -pKEY -v

# Backup BitLocker metadata (advanced)
sudo dd if=/dev/sda2 of=/mnt/backup/bitlocker-metadata.bin bs=1M count=10
```
