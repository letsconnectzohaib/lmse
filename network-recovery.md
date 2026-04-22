# Network-Based Recovery

Recover data over network when local storage is insufficient or unavailable.

---

## Why Network Recovery?

- Destination drive is smaller than source
- Multiple machines to recover
- No available USB ports/storage
- Remote location assistance
- Continuous backup during recovery

---

## SSH/SCP - Simple File Transfer

### From Source Machine (BlackArch)

```bash
# Start SSH server
sudo systemctl start sshd

# Set root password for recovery
sudo passwd root

# Check IP address
ip addr show
```

### From Destination Machine

```bash
# Pull files from source
scp -r root@192.168.1.100:/mnt/windows/Users/Name/Documents ./recovery/

# Or rsync for resume capability
rsync -avz --progress root@192.168.1.100:/mnt/windows/Users/ ./recovery/
```

### Reverse Direction (Push)

```bash
# From source machine, push to destination
scp -r /mnt/windows/Users/Name/Documents root@192.168.1.50:/mnt/backup/
```

---

## NFS - Network File System

### Setup NFS Server (Destination)

```bash
# Install NFS
sudo pacman -S nfs-utils

# Create export directory
sudo mkdir -p /srv/nfs/recovery
sudo chmod 777 /srv/nfs/recovery

# Configure exports
sudo tee /etc/exports << 'EOF'
/srv/nfs/recovery 192.168.1.0/24(rw,no_root_squash,sync,no_subtree_check)
EOF

# Start NFS
sudo systemctl start nfs-server
sudo systemctl start rpcbind
sudo exportfs -ra
```

### Mount NFS (Source - BlackArch)

```bash
# Install NFS client
sudo pacman -S nfs-utils

# Mount remote share
sudo mkdir -p /mnt/remote
sudo mount -t nfs 192.168.1.50:/srv/nfs/recovery /mnt/remote

# Copy files
sudo cp -r /mnt/windows/Users/Name/* /mnt/remote/
# Or use rsync
rsync -avh --progress /mnt/windows/Users/Name/Documents /mnt/remote/
```

---

## Samba - Windows File Sharing

### Setup Samba Server (Destination)

```bash
# Install Samba
sudo pacman -S samba

# Create share directory
sudo mkdir -p /srv/samba/recovery
sudo chmod 777 /srv/samba/recovery

# Configure smb.conf
sudo tee -a /etc/samba/smb.conf << 'EOF'
[recovery]
path = /srv/samba/recovery
writable = yes
browsable = yes
guest ok = yes
create mask = 0777
directory mask = 0777
EOF

# Start Samba
sudo systemctl start smb nmb
```

### Mount Samba Share (BlackArch)

```bash
# Install CIFS utils
sudo pacman -S cifs-utils

# Mount Windows share
sudo mkdir -p /mnt/samba
sudo mount -t cifs //192.168.1.50/recovery /mnt/samba -o guest

# Or with credentials
sudo mount -t cifs //192.168.1.50/recovery /mnt/samba \
  -o username=backupuser,password=secret

# Copy files
rsync -avh /mnt/windows/Users/Name/Documents /mnt/samba/
```

---

## Netcat - Direct Transfer

### Fast Transfer Without Encryption

**On destination (receiver):**
```bash
# Prepare to receive
nc -l 1234 | dd of=/mnt/backup/image.img

# Or compress on the fly
nc -l 1234 | gzip > /mnt/backup/image.img.gz
```

**On source (sender):**
```bash
# Send partition
dd if=/dev/sda2 | nc 192.168.1.50 1234

# With compression
dd if=/dev/sda2 | gzip | nc 192.168.1.50 1234

# With progress
dd if=/dev/sda2 status=progress | nc 192.168.1.50 1234
```

---

## Rsync Over SSH

### Best for Resume Capability

```bash
# Basic rsync over SSH
rsync -avz --progress /mnt/windows/Users/ root@192.168.1.50:/backup/

# With compression and partial resume
rsync -avzP --compress-level=9 \
  /mnt/windows/Users/Name/Documents \
  root@192.168.1.50:/backup/

# Exclude patterns
rsync -avz --progress \
  --exclude='*.tmp' \
  --exclude='Temp' \
  /mnt/windows/Users/Name/ \
  root@192.168.1.50:/backup/
```

### Rsync Daemon Mode

**Server side:**
```bash
# /etc/rsyncd.conf
[backup]
path = /srv/backup
read only = no
uid = root
gid = root

# Run
cat /etc/rsyncd.conf | sudo rsync --daemon --no-detach --config=-
```

**Client side:**
```bash
rsync -avz /mnt/windows/Users/ rsync://192.168.1.50/backup/
```

---

## Cloud Upload During Recovery

### Rclone (Multiple Cloud Providers)

```bash
# Install rclone
sudo pacman -S rclone

# Configure
rclone config
# Follow prompts to add Google Drive, S3, etc.

# Upload files
rclone copy /mnt/windows/Users/Name/Documents gdrive:Recovery/

# Sync (mirrors, deletes extra files)
rclone sync /mnt/windows/Users/Name/ gdrive:Recovery/

# With progress
rclone copy --progress /mnt/windows/Users/ gdrive:Recovery/
```

### AWS S3 Direct Upload

```bash
# Install AWS CLI
sudo pacman -S aws-cli

# Configure credentials
aws configure

# Upload
aws s3 sync /mnt/windows/Users/ s3://my-recovery-bucket/

# With storage class (cheaper)
aws s3 sync /mnt/windows/Users/ s3://my-recovery-bucket/ \
  --storage-class DEEP_ARCHIVE
```

### FTP/SFTP Upload

```bash
# Using lftp
sudo pacman -S lftp

# Upload directory
lftp -u username,password ftp.example.com -e "mirror -R /mnt/windows/Users /remote/path; bye"

# SFTP (SSH)
lftp sftp://root:password@192.168.1.50 -e "mirror -R /mnt/windows/Users /backup; bye"
```

---

## Live Network Boot

### PXE Boot Recovery (Recover Multiple Machines)

**Server Setup (BlackArch with NFS):**
```bash
# Install required packages
sudo pacman -S dnsmasq syslinux nfs-utils

# Configure TFTP/DHCP
sudo tee /etc/dnsmasq.conf << 'EOF'
interface=eth0
dhcp-range=192.168.1.100,192.168.1.200,12h
dhcp-boot=pxelinux.0
tftp-root=/srv/tftp
enable-tftp
EOF

# Setup PXE files
sudo mkdir -p /srv/tftp
sudo cp /usr/lib/syslinux/bios/pxelinux.0 /srv/tftp/
sudo cp /usr/lib/syslinux/bios/{menu.c32,ldlinux.c32} /srv/tftp/

# Create PXE menu
sudo mkdir -p /srv/tftp/pxelinux.cfg
sudo tee /srv/tftp/pxelinux.cfg/default << 'EOF'
DEFAULT menu.c32
PROMPT 0
MENU TITLE Recovery Boot
TIMEOUT 100

LABEL blackarch
MENU LABEL BlackArch Recovery
KERNEL blackarch/vmlinuz
APPEND initrd=blackarch/initrd.img blackarch.nox ip=dhcp
EOF

# Copy BlackArch kernel/initrd
sudo mkdir -p /srv/tftp/blackarch
sudo cp /boot/vmlinuz-linux /srv/tftp/blackarch/vmlinuz
sudo cp /boot/initramfs-linux.img /srv/tftp/blackarch/initrd.img

# Start services
sudo systemctl start dnsmasq
sudo systemctl start rpcbind nfs-server

# Export recovery directory
sudo tee /etc/exports << 'EOF'
/srv/recovery 192.168.1.0/24(rw,no_root_squash)
EOF
sudo exportfs -ra
```

**Client:**
- Boot from network (F12 or BIOS setting)
- Gets BlackArch via PXE
- Can mount NFS share for saving recovered data

---

## Network Troubleshooting

### Check Connectivity

```bash
# Check IP
ip addr show

# Check gateway
ip route show

# Test connection
ping 192.168.1.1
ping 8.8.8.8

# Check DNS
nslookup google.com
```

### Fix Network Issues

```bash
# DHCP not working? Set static IP
sudo ip addr add 192.168.1.100/24 dev eth0
sudo ip route add default via 192.168.1.1
sudo tee /etc/resolv.conf << 'EOF'
nameserver 8.8.8.8
nameserver 8.8.4.4
EOF

# Restart networking
sudo systemctl restart systemd-networkd
```

### Transfer Interrupted?

```bash
# rsync resumes automatically
rsync -avzP /source/ user@host:/dest/

# If using dd + netcat, use ddrescue
# It can resume with log file
ddrescue -r3 /dev/sda - logfile | nc host port
```

---

## Bandwidth Optimization

### Compress During Transfer

```bash
# Compress with pigz (parallel gzip)
tar -cf - /mnt/windows/Users | pigz | nc 192.168.1.50 1234

# Receiver
nc -l 1234 | pigz -d | tar -xf - -C /backup/
```

### Limit Bandwidth

```bash
# rsync with bandwidth limit (100KB/s)
rsync -avz --bwlimit=100 /source/ host:/dest/

# scp limit (Kbit/s)
scp -l 800 /file user@host:/dest/  # 800 Kbit/s = 100KB/s

# trickle for any command
trickle -u 1000 nc -l 1234  # limit upload to 1MB/s
```

---

## Quick Commands

```bash
# Quick SSH file pull
scp -r root@IP:/path ./local/

# Quick NFS mount
sudo mount -t nfs IP:/share /mnt/nfs

# Quick SMB mount
sudo mount -t cifs //IP/share /mnt/smb -o guest

# Quick netcat transfer (receiver)
nc -l 1234 > file

# Quick netcat transfer (sender)
cat file | nc IP 1234
```
