# Windows Recovery Environment Guide

Boot repair and data recovery using Windows Recovery USB.

---

## Create Windows Recovery USB

1. Download Windows Media Creation Tool from Microsoft
2. Create bootable USB (8GB+ required)
3. Boot corrupted PC from USB
4. Select **Repair your computer** → **Troubleshoot** → **Command Prompt**

---

## Check BitLocker Status

```cmd
manage-bde -status
```

| Status | Meaning |
|--------|---------|
| `Protection: On` | BitLocker encrypted |
| `Protection: Off` | Not encrypted |
| `Locked` | Drive locked, needs key |
| `Unlocked` | Drive accessible |

---

## Fix Boot Issues

### MBR Repair (Legacy BIOS)

```cmd
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
```

### UEFI Repair

```cmd
bootrec /fixmbr
bootrec /fixboot
bootrec /rebuildbcd
bcdboot C:\Windows /f UEFI
```

---

## Disk Check and Repair

```cmd
chkdsk C: /f /r
```

| Parameter | Function |
|-----------|----------|
| `/f` | Fix errors |
| `/r` | Recover bad sectors |
| `/x` | Force unmount first |

---

## Access Files in Recovery Mode

```cmd
# List all drives
wmic logicaldisk get name

# Check what's on C:
dir C:\

# Check Users folder
dir C:\Users

# Copy files to USB (D: is USB)
xcopy "C:\Users\YourName\Documents" D:\Backup\ /E /I /H
```

---

## Unlock BitLocker from Recovery

```cmd
# If you have recovery key
manage-bde -unlock C: -rp YOUR-RECOVERY-KEY

# If you have password
manage-bde -unlock C: -pw

# Disable BitLocker (after unlocking)
manage-bde -off C:
```

---

## System Restore Commands

```cmd
# List restore points
rstrui.exe /offline:C:\Windows=G:\Windows

# Run system restore
rstrui.exe
```

---

## Registry Repair

```cmd
# Load registry hive
reg load HKLM\Temp C:\Windows\System32\config\SOFTWARE

# Export for backup
reg export HKLM\Temp C:\temp-software.reg

# Unload when done
reg unload HKLM\Temp
```

---

## Reset Windows (Keep Files)

From Recovery menu:
1. **Troubleshoot** → **Reset this PC**
2. Choose **Keep my files**
3. Select target OS
4. Wait for completion

---

## Quick Reference

| Problem | Command |
|---------|---------|
| Boot loop | `bootrec /rebuildbcd` |
| Missing OS | `bcdboot C:\Windows` |
| Corrupted disk | `chkdsk C: /f /r` |
| Locked BitLocker | `manage-bde -unlock C: -rp KEY` |
| Copy files | `xcopy source dest /E /I` |

---

## Third-Party Recovery Tools

| Tool | Function |
|------|----------|
| Macrium Reflect | Disk imaging |
| Acronis True Image | Backup/Recovery |
| Hiren's BootCD | All-in-one toolkit |
| Medicat USB | Multiboot recovery |
| DMDE | Partition recovery |
