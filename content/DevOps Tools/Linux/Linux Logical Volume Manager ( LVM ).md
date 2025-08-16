---
longform:
  format: single
  title: Linux Logical Volume Manager ( LVM )
title: Linux Logical Volume Manager ( LVM )
---

## Concept of LVM

**Logical Volume Manager (LVM)** is a device-mapper framework in Linux that enables flexible and dynamic management of disk storage. LVM abstracts physical storage devices and allows administrators to create "logical volumes," which act like partitions but offer greater flexibility. This is particularly useful for resizing, moving, or managing disks without service disruption.[wikipedia+3](https://en.wikipedia.org/wiki/Logical_Volume_Manager_\(Linux\))

LVM structure uses three key components:

- **Physical Volumes (PV):** Actual storage devices (disks, partitions).
- **Volume Groups (VG):** Pool of aggregated physical volumes.
- **Logical Volumes (LV):** Virtual partitions from the VG, used by file systems or applications.
- **Physical Extents (PE):** Smallest allocatable unit within PV.

---

## Features of LVM

- **Flexibility:** Easily resize, move, or create logical volumes as storage requirements change.
    
- **Dynamic resizing:** Logical volumes and volume groups can be grown or shrunk online.
    
- **No downtime:** Add/remove disks, extend/reduce storage—all without taking the system offline.
    
- **Pooling storage:** Combine multiple disks/partitions into one large storage pool.
    
- **Snapshots:** Create point-in-time copies of logical volumes for backup/restore.
    
- **Performance options:** Use striping for speed or mirroring for redundancy.
    
- **Thin provisioning:** Allocate storage "on demand" rather than upfront.
    
- **Convenient admin commands:** Rich set of CLI tools for management (e.g. pvcreate, vgcreate, lvcreate, pvmove).

---

## LVM Architecture Diagram

```
[ Storage Devices (PV) ]
        |        |
      [Partition1] [Partition2]
        |        |
   --> PV1      PV2
        \      /
       [ Volume Group (VG1) ]
              |
        [ Logical Volume (LV1) ]
        [ Logical Volume (LV2) ]
              |
          [Filesystem]
              |
         [Application/Data]
```

- Physical disks or disk partitions are initialized as Physical Volumes (PV).
    
- Multiple PVs are grouped into a Volume Group (VG). The VG acts as a storage pool.
    
- Logical Volumes (LV) are created from the space provided by VG.
    
- File systems are created on the LV, which then store application data.

---

## Common LVM Administrative Steps

1. **Scan and prepare disks**
    
    - Identify storage devices: `lvmdiskscan`
        
    - Partition disks (if needed) and set type to Linux LVM (8e): `fdisk`
        
2. **Create Physical Volume (PV):**

```
pvcreate /dev/sdb1
pvcreate /dev/sdc1
```

3. **Create Volume Group (VG):**

```
vgcreate myvg /dev/sdb1 /dev/sdc1

```

4. **Create Logical Volume (LV):**

```
lvcreate -L 20G -n mylv myvg
```

5. **Format LV and mount:**

```
mkfs.ext4 /dev/myvg/mylv
mount /dev/myvg/mylv /mnt/data
```

6. **Extend Logical Volume (Online):**

```
lvextend -L +10G /dev/myvg/mylv
resize2fs /dev/myvg/mylv
```

7. **Create Snapshot (for backup):**

```
lvcreate -s -n mysnap -L 2G /dev/myvg/mylv
```
8. **Move data to another disk:**

  ```
  pvmove /dev/sdb1
```

9. **Remove a Physical Volume from Volume Group:**

```
  vgreduce myvg /dev/sdb1

```

---

## Interview-Ready Summary

- **LVM is ideal for modern systems requiring flexible storage management.**
    
- **Key benefits:** Resize disks, split/merge allocations, use RAID-like striping/mirroring, enable snapshots and backups, and manage large disk farms with zero downtime.[golinuxcloud+3](https://www.golinuxcloud.com/overview-lvm-in-linux/)
    
- **Administration tools and steps are intuitive and allow efficient storage configuration.**
    
- **Diagram understanding shows how storage is abstracted and mapped for applications.**

---

LVM's abstraction turns physical storage management into a set of simple, powerful logical operations—making it a crucial concept for Linux system administrators.