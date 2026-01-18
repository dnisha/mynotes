---
longform:
  format: single
  title: Standard Directory Structure
title: Standard Directory Structure
---

```bash

/           - Root directory
├── /bin    - Essential user binaries (ls, cp, mkdir)
├── /sbin   - System administration binaries
├── /etc    - Configuration files
├── /home   - User home directories
├── /root   - Root user's home
├── /usr    - User programs and data
├── /var    - Variable data (logs, spool files)
├── /tmp    - Temporary files
├── /dev    - Device files
├── /proc   - Process information
├── /boot   - Boot loader files
└── /lib    - Shared libraries

```

### Important Path Concepts

- **Absolute Path**: Starts from root (`/home/user/file.txt`)
- **Relative Path**: Relative to current directory (`./file.txt` or `../folder/`)
- **Special Directories**:
    - `.` - Current directory
        
    - `..` - Parent directory
        
    - `~` - Home directory
        
    - `-` - Previous directory