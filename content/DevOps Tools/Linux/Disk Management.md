---
longform:
  format: single
  title: Disk Management
title: Disk Management
---
## **Partitioning**

```bash

# fdisk - Traditional partition tool
sudo fdisk -l              # List partitions
sudo fdisk /dev/sda        # Interactive partition editor

# parted - Advanced partition tool
sudo parted -l
sudo parted /dev/sda
# In parted:
# print                   # Show partitions
# mkpart primary ext4 1GB 10GB  # Create partition
# set 1 boot on          # Set boot flag

# gdisk - GPT partition tool
sudo gdisk /dev/sda

```

## **Filesystem Operations**

```bash

# Create filesystem
sudo mkfs.ext4 /dev/sda1
sudo mkfs.xfs /dev/sda2
sudo mkfs.ntfs /dev/sda3

# Mount filesystems
sudo mount /dev/sda1 /mnt
sudo mount -t ext4 /dev/sda1 /mnt
sudo mount -o ro /dev/sda1 /mnt  # Read-only

# Unmount
sudo umount /mnt
sudo umount /dev/sda1

# Auto-mount (/etc/fstab)
# /dev/sda1 /mnt ext4 defaults 0 0

```

## **Disk Usage Analysis**

```bash

# df - Disk Free
df -h                      # Human readable
df -i                      # Inode usage
df -T                      # Show filesystem types

# du - Disk Usage
du -sh folder/             # Summary human readable
du -h --max-depth=1        # One level deep
du -ah folder/             # All files with sizes

# Find large files
find / -type f -size +100M  # Files >100MB
find / -type f -size +1G    # Files >1GB

```

# **Mounting & Unmounting EBS Volumes in EC2**

## **Prerequisites Check**

**List available disks:**

```bash

lsblk
# or
sudo fdisk -l
# or for AWS-specific info
sudo lsblk -f

```

### **SSH into EC2 and prepare volume:**

```bash

# List block devices
lsblk

# Create filesystem (if new/empty volume)
sudo mkfs -t ext4 /dev/xvdf
# For xfs:
sudo mkfs.xfs /dev/xvdf

# Create mount directory
sudo mkdir /mnt/data

# Mount the volume
sudo mount /dev/xvdf /mnt/data

# Verify
df -h
mount | grep xvdf

```

### **Make mount permanent (add to fstab):**

```bash

# Get UUID
sudo blkid /dev/xvdf

# Backup fstab
sudo cp /etc/fstab /etc/fstab.backup

# Edit fstab
sudo nano /etc/fstab
# Add line:
UUID=your-uuid-here   /mnt/data   ext4   defaults,nofail   0   2

# Test fstab (without mounting)
sudo mount -a

```

## **Unmounting EBS Volume**

### **Unmount properly:**

```bash

# Check what's using the mount
sudo lsof /mnt/data
# or
sudo fuser -m /mnt/data

# Unmount
sudo umount /mnt/data
# Force unmount if busy
sudo umount -l /mnt/data  # lazy unmount

```