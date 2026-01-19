---
longform:
  format: single
  title: File Permissions
title: File Permissions
---
# Linux File Permissions Explained

File permissions in Linux are a fundamental security feature that controls access to files and directories. Here's a comprehensive explanation:

## Basic Permission System

Linux uses three types of permissions for each file and directory:

1. **Read (r)** - Allows viewing/reading file contents or listing directory contents
2. **Write (w)** - Allows modifying file contents or adding/removing files in directories
3. **Execute (x)** - Allows running a file as a program or accessing contents of a directory

## Permission Classes

Permissions are assigned to three classes of users:

1. **Owner (u)** - The user who owns the file
2. **Group (g)** - Members of the file's group
3. **Others (o)** - All other users

## Viewing Permissions

Use `ls -l` to view permissions:

```
-rwxr-xr-- 1 user group 2048 Jan 1 10:00 file.txt
```

The permission string (`-rwxr-xr--`) breaks down as:
- First character: file type (`-` for regular file, `d` for directory)
- Next 3: owner permissions (`rwx`)
- Next 3: group permissions (`r-x`)
- Last 3: others permissions (`r--`)

## Numeric (Octal) Representation

Each permission has a numeric value:
- Read (r) = 4
- Write (w) = 2
- Execute (x) = 1

These are added together for each class:
- `rwxr-xr--` becomes 754 (7 for owner, 5 for group, 4 for others)

## Changing Permissions

Use `chmod` to change permissions:

1. Symbolic mode:
   ```bash
   chmod u+x file.txt      # Add execute for owner
   chmod g-w file.txt      # Remove write for group
   chmod o=r file.txt      # Set others to read-only
   chmod a+x file.txt      # Add execute for all (a = all)
   ```

2. Numeric mode:
   ```bash
   chmod 755 file.txt      # rwxr-xr-x
   chmod 644 file.txt      # rw-r--r--
   ```

## Changing Ownership

Use `chown` and `chgrp` to change owner and group:
```bash
chown user file.txt        # Change owner
chown user:group file.txt  # Change both owner and group
chgrp group file.txt       # Change group only
```

# Special Permissions in Linux: SUID, SGID, and Sticky Bit

## 1. **Set User ID (SUID)**

### What it does:

- When set on an executable file, the program runs with the **owner's privileges** instead of the user's who executes it.
- Represented by `s` in the execute position for owner.

### Numeric value: `4000`

### Example:

```bash

# Check current permissions
ls -l /usr/bin/passwd
# Typically shows: -rwsr-xr-x 1 root root

# Set SUID (using symbolic notation)
chmod u+s /path/to/executable

# Set SUID (using numeric notation)
chmod 4755 /path/to/executable

```

**What 4755 means:**

- `4` = SUID bit
- `755` = rwxr-xr-x (owner: rwx, group: r-x, others: r-x)

**Real-world example:**

```bash

# /usr/bin/passwd needs SUID because:
# - It's owned by root
# - Regular users need to modify /etc/shadow (which only root can normally write to)
# - With SUID, when a user runs passwd, it runs with root privileges temporarily

# Check it:
ls -l /usr/bin/passwd
# Output: -rwsr-xr-x 1 root root 63960 Feb  7  2020 /usr/bin/passwd
# The 's' in owner's execute position indicates SUID

```

## 2. **Set Group ID (SGID)**

### What it does:

1. **On executables**: Runs with the **group's privileges** instead of the user's group.

2. **On directories**: New files created in the directory inherit the directory's group ownership.

### Numeric value: `2000`

### Example:

```bash

# For executables (like SUID but for group)
chmod 2755 /path/to/executable

# For directories
chmod 2770 /shared-directory

```

**What 2755 means:**

- `2` = SGID bit
- `755` = rwxr-xr-x

### Directory example:

```bash

# Create a shared directory for a team
mkdir /shared
chgrp developers /shared
chmod 2770 /shared

# Now, any file created in /shared will have 'developers' as group
ls -ld /shared
# Output: drwxrws--- 2 root developers 4096 Dec 10 10:00 /shared
# The 's' in group's execute position indicates SGID

# Test it:
touch /shared/testfile
ls -l /shared/testfile
# Output: -rw-r--r-- 1 youruser developers 0 Dec 10 10:00 testfile
# Note: group is 'developers' (inherited from directory), not your primary group!

```

##  **Sticky Bit**

### What it does:

- **On directories**: Users can only delete/rename files they own, even if they have write permission to the directory.

- Common on `/tmp` and `/var/tmp` directories.

### Numeric value: `1000`

### Example:

```bash

# Set sticky bit on a directory
chmod 1777 /tmp

# Or using symbolic notation
chmod +t /directory

```

**What 1777 means:**

- `1` = Sticky bit
- `777` = rwxrwxrwx (full permissions for all)

### Practical example:

```bash

# Check /tmp directory
ls -ld /tmp
# Output: drwxrwxrwt 10 root root 4096 Dec 10 10:00 /tmp
# The 't' in others' execute position indicates sticky bit

# How it works:
# - Everyone has rwx permissions to /tmp
# - User1 creates file1 in /tmp
# - User2 can read/write file1 (if permissions allow)
# - User2 CANNOT delete file1 (only User1 or root can)

```

# **umask (User File Creation Mask)**

## **What is umask?**

- A mask that **subtracts** permissions from the maximum default permissions
- It's **NOT** a permission value itself, but a filter that removes permissions
- Controls default permissions for newly created files and directories
- Expressed in **octal** notation (like 022, 027, 077)

---

## **How umask works:**

### **Default Maximum Permissions:**

1. **Files**: `666` (rw-rw-rw-)
    - Files aren't executable by default for security
2. **Directories**: `777` (rwxrwxrwx)

### **The Calculation:**

**Final Permission = Maximum Permission - umask**

**Important:** It's a **bitwise AND with complement** operation, not simple subtraction:

- `permission = maximum_permission & ~umask`

## **Common umask values:**

### **1. umask 022 (Most common default)**

```bash

umask 022

# For files: 666 - 022 = 644
touch newfile.txt
ls -l newfile.txt  # -rw-r--r-- (644)

# For directories: 777 - 022 = 755
mkdir newdir
ls -ld newdir  # drwxr-xr-x (755)

```

### **2. umask 027 (More restrictive)**

```bash

umask 027

# For files: 666 - 027 = 640
touch file1.txt
ls -l file1.txt  # -rw-r----- (640)

# For directories: 777 - 027 = 750  
mkdir dir1
ls -ld dir1  # drwxr-x--- (750)

```

## **Viewing and Setting umask:**

### **1. Check current umask:**

```bash

umask          # Numeric form: 0022
umask -S       # Symbolic form: u=rwx,g=rx,o=rx

```