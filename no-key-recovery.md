# Recovery Without Keys or Passwords

Last resort methods when you don't have BitLocker keys or Windows passwords.

---

## The Reality

| Encryption | Recoverable Without Key? |
|------------|--------------------------|
| BitLocker | ❌ No (AES encryption) |
| Windows Password Only | ✅ Yes (easy) |
| Corrupted NTFS | ⚠️ Partial (PhotoRec) |

---

## Method 1: PhotoRec (Raw File Recovery)

Recovers files by scanning for file signatures. **No folder structure, no filenames.**

### Install:

```bash
# Arch/BlackArch
sudo pacman -S testdisk

# Ubuntu/Debian
sudo apt install testdisk
```

### Run PhotoRec:

```bash
sudo photorec /dev/sda2
```

### Steps:

1. Select partition → `Proceed`
2. `File Opt` → Select file types:
   - `[X] jpg` - Photos
   - `[X] pdf` - Documents
   - `[X] doc` - Word files
   - `[X] xls` - Excel files
   - `[X] mp4` - Videos
   - `[X] txt` - Text files
3. `Search` → Select `Other` (for unknown/encrypted filesystem)
4. Choose destination folder (external USB)
5. Let it run (takes hours)

### Results:

```
/mnt/usb/recup_dir.1/
  f0000001.jpg
  f0000002.pdf
  f0000003.docx
  ...
```

You'll manually sort through thousands of files.

---

## Method 2: TestDisk (Partition Recovery)

For lost/corrupted partitions, not encrypted ones:

```bash
sudo testdisk /dev/sda
```

1. Create log → Select disk → Intel/PC partition
2. `Analyse` → `Quick Search` → `Deeper Search`
3. Select found partition → `Write`

---

## Method 3: ddrescue (Create Image First)

Always work on a copy if drive is failing:

```bash
# Install
sudo pacman -S ddrescue

# Create image
sudo ddrescue /dev/sda2 /mnt/usb/disk-image.img /mnt/usb/rescue.log

# Mount image
sudo losetup -fP /mnt/usb/disk-image.img
sudo mount /dev/loop0p1 /mnt/copy
```

---

## Method 4: Strings (Extract Text)

Quick and dirty text extraction:

```bash
# Extract readable strings
sudo strings /dev/sda2 > /mnt/usb/recovered-text.txt

# Search for specific content
sudo strings /dev/sda2 | grep -i "important document"
```

---

## Method 5: Forensic Carving

Advanced tools for specific file types:

### Foremost:

```bash
sudo pacman -S foremost
sudo foremost -i /dev/sda2 -o /mnt/usb/output/ -t jpg,pdf,doc
```

### Scalpel:

```bash
sudo pacman -S scalpel
sudo scalpel /dev/sda2 -o /mnt/usb/scalpel-output/
```

---

## Method 6: Windows Password Reset (Not BitLocker)

If just Windows password locked:

### Using chntpw:

```bash
sudo pacman -S chntpw

# Mount Windows partition
sudo mount /dev/sda2 /mnt/windows

# Edit SAM database
cd /mnt/windows/Windows/System32/config
sudo chntpw -u YourUsername SAM

# Options:
# 1 - Clear password
# 2 - Set new password
# q - Quit and save
```

### Using Hiren's BootCD:

1. Boot from USB
2. **Mini Windows XP** or **Windows 10 PE**
3. Open **NT Password Edit**
4. Select SAM file → Clear password
5. Save and reboot

---

## Expected Recovery Rates

| Method | Success Rate | Quality |
|--------|--------------|---------|
| PhotoRec | 40-70% | No names/folders |
| Foremost | 30-60% | No names/folders |
| Strings | 10-20% | Text only |
| Professional | 60-90% | $$$ |

---

## Professional Data Recovery

When all else fails:

| Service | Cost | Timeline |
|---------|------|----------|
| DriveSavers | $500-3000 | 2-5 days |
| Ontrack | $400-2500 | 1-7 days |
| Local shops | $200-800 | 3-10 days |

**They can:**
- Chip-off recovery (remove flash chips)
- Clean room platter recovery
- Electron microscope reading (extreme cases)

**They cannot:**
- Crack BitLocker encryption

---

## Quick Checklist

1. ✅ Confirm it's BitLocker (not just password)
2. ✅ Try PhotoRec on external drive
3. ✅ Check Microsoft account one more time
4. ✅ Check old emails for recovery key
5. ✅ Try Acronis backup if available
6. ⏳ Decide if data worth professional cost
