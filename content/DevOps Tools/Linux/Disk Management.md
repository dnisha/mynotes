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

## **Create Partitions**

```bash

#!/bin/bash

# This script creates partitions on /dev/xvdf disk in EC2 instance

# Update package list to ensure system is up-to-date
echo "Updating package list..."
sudo apt update -y

# Display all block devices to identify available disks
echo "Checking available block devices..."
lsblk

# Start parted interactive session on /dev/xvdf disk
# parted is a disk partitioning utility
echo "Starting parted on /dev/xvdf..."
sudo parted /dev/xvdf <<EOF

# Display current partition table (should show no partitions initially)
# The print command shows disk information and existing partitions
print

# Create a new GPT (GUID Partition Table) disk label
# GPT is modern replacement for MBR, supports disks >2TB and more partitions
mklabel gpt

# Display partition table after creating GPT label
# This confirms GPT label was created successfully
print

# Create first partition (part1)
# - Start at 1MB (aligns with modern standards)
# - End at 500MB (creates ~499MB partition)
# - Using ext2 as placeholder filesystem type (can be changed later)
mkpart part1 ext2 1M 500M

# Display partition table after creating first partition
# Shows partition 1 was created successfully
print

# Create second partition (part2)
# - Start at 500MB (immediately after first partition)
# - End at 1024MB (creates ~524MB partition)
mkpart part2 ext2 500M 1024M

# Display final partition table
# Shows both partitions created successfully
print

# Exit parted interactive mode
quit
EOF

# partprobe informs the OS of partition table changes
# This reloads partition table without requiring reboot
echo "Reloading partition table..."
sudo partprobe /dev/xvdf

# Display block devices to verify partitions are visible to the OS
echo "Verifying partitions were created..."
lsblk

# Check detailed partition information using parted
echo "Displaying detailed partition information..."
sudo parted /dev/xvdf print

# Format partitions with ext4 filesystem (commonly used in Linux)
# mkfs.ext4 creates ext4 filesystem on specified partition
echo "Formatting partitions with ext4 filesystem..."
sudo mkfs.ext4 /dev/xvdf1
sudo mkfs.ext4 /dev/xvdf2

# Create mount points (directories where partitions will be accessed)
# -p flag creates parent directories if they don't exist
echo "Creating mount directories..."
sudo mkdir -p /mnt/part1
sudo mkdir -p /mnt/part2

# Mount partitions to mount points
# mount attaches filesystem to directory tree
echo "Mounting partitions..."
sudo mount /dev/xvdf1 /mnt/part1
sudo mount /dev/xvdf2 /mnt/part2

# Display disk usage with human-readable format and filesystem type
# Verifies partitions are mounted correctly
echo "Verifying mounts..."
df -hT

# Add partitions to /etc/fstab for automatic mounting at boot
# - defaults: Use default mount options
# - nofail: Don't halt boot if device not found (important for EC2)
# - 0 2: dump and fsck order values
echo "Adding partitions to /etc/fstab for automatic mounting..."
echo "/dev/xvdf1 /mnt/part1 ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab
echo "/dev/xvdf2 /mnt/part2 ext4 defaults,nofail 0 2" | sudo tee -a /etc/fstab

# Test fstab configuration by mounting all filesystems
# mount -a mounts all filesystems mentioned in fstab
echo "Testing fstab configuration..."
sudo mount -a

# Final verification of mounts
echo "Final mount verification..."
df -hT | grep -E "(Filesystem|/dev/xvdf)"

echo "Partition creation and setup completed successfully!"

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