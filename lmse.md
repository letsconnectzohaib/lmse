# Disk & Boot Troubleshooting Guide

A step-by-step guide for diagnosing disk encryption, filesystem issues, and boot problems.

---

## PART 1 — Boot into Linux (Live USB preferred)

When you're inside Linux, open terminal and run:

### 1. Check if disk is detected

```bash
lsblk
```

**Look for:**
- `sda` or `nvme0n1`
- partitions like `sda1`, `sda2`

### 2. Check filesystem type (VERY IMPORTANT)

```bash
lsblk -f
```

**Tell me what shows under FSTYPE:**

| Output | Meaning |
|--------|---------|
| `ntfs` | normal Windows |
| `crypto_LUKS` | Linux encryption |
| `unknown` / blank | damaged or BitLocker |
| `vfat` (small partition) | EFI boot |

### 3. Try mounting Windows manually

> Replace partition name if needed

```bash
sudo mkdir /mnt/test
sudo mount /dev/sda2 /mnt/test
```

**Result possibilities:**

| Result | Meaning |
|--------|---------|
| ✅ Mounts successfully | `ls /mnt/test` → You'll see Windows files = **GOOD (no encryption)** |
| 🔐 Asks for password | **Encryption detected (very important)** |
| ❌ Error | Run: `sudo ntfsfix /dev/sda2` |

### 4. Check disk health (quick)

```bash
dmesg | grep -i error
```

**Look for:**
- `I/O errors` → hardware issue
- `NTFS errors` → filesystem problem

---

## PART 2 — BIOS CHECK (super important)

Go into BIOS and note:

| Setting | Value |
|---------|-------|
| Boot Mode | UEFI or Legacy |
| Secure Boot | ON / OFF |
| SSD Visibility | Is SSD visible? |

**Tell me exactly what these are.**

---

## PART 3 — Windows Recovery (USB)

Boot Windows USB → Command Prompt

### 5. Check BitLocker status

```cmd
manage-bde -status
```

**Look for:**
- `Locked` / `Unlocked`
- `Protection ON`

### 6. Try disk check

```cmd
chkdsk C: /f /r
```

### 7. Try boot repair

```cmd
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

### 8. If UEFI system (important)

```cmd
bcdboot C:\Windows /f UEFI
```

---

## WHAT TO REPORT BACK

Just send:

### From Linux:
- Output of: `lsblk`
- Output of: `lsblk -f`
- What happens when mounting

### From BIOS:
- UEFI or Legacy?
- Secure Boot ON/OFF

### From Windows CMD:
- Output of: `manage-bde -status`
