---
longform:
  format: single
  title: Shell Scripting
title: Shell Scripting
---
## **Script Basics**

```bash

#!/bin/bash
# This is a comment
echo "Hello World!"

```

## **Variables**

```bash

# Variable assignment
name="John"
age=25

# Using variables
echo "Name: $name"
echo "Age: ${age}"

# Command substitution
current_date=$(date)
backtick_date=`date`

# Read input
read -p "Enter name: " username
echo "Hello, $username"

```

## **Output & Redirection**

```bash

# echo vs printf
echo "Hello"             # Automatic newline
printf "Hello\n"         # Manual newline control

# Redirection
command > file           # Overwrite output
command >> file          # Append output
command < file           # Input from file
command 2> error.log     # Redirect stderr
command > output.log 2>&1  # Redirect both stdout & stderr
command &> file          # Redirect both (bash)

# Pipes
command1 | command2      # Pipe output
ls -la | grep "file" | wc -l

```

## **Conditional Statements**

```bash

# if-elif-else
if [ condition ]; then
    commands
elif [ condition ]; then
    commands
else
    commands
fi

# Comparison operators
# Numeric: -eq, -ne, -lt, -gt, -le, -ge
# String: =, !=, -z (empty), -n (not empty)
# File: -f (file exists), -d (directory exists), -r (readable)

# Example
if [ $num -eq 10 ]; then
    echo "Number is 10"
fi

if [ -f "/path/to/file" ]; then
    echo "File exists"
fi
```

## **Loops**

```bash

# For loop
for i in 1 2 3 4 5; do
    echo "Number: $i"
done

for file in *.txt; do
    echo "Processing $file"
done

for ((i=0; i<10; i++)); do
    echo "Count: $i"
done

# While loop
counter=1
while [ $counter -le 5 ]; do
    echo "Counter: $counter"
    ((counter++))
done

# Reading file line by line
while read line; do
    echo "Line: $line"
done < file.txt

# Until loop
counter=1
until [ $counter -gt 5 ]; do
    echo "Counter: $counter"
    ((counter++))
done

# Loop control
for i in {1..10}; do
    if [ $i -eq 5 ]; then
        break           # Exit loop
    fi
    if [ $i -eq 3 ]; then
        continue        # Skip iteration
    fi
    echo "Number: $i"
done

```

## **Functions**

```bash

# Define function
function_name() {
    commands
    return value
}

# Or
function function_name {
    commands
}

# Example
greet() {
    echo "Hello, $1!"
    return 0
}

# Call function
greet "Alice"

# Capture return value
greet "Bob"
echo "Exit status: $?"
```

## **Advanced Scripting**

```bash

# Arrays
fruits=("apple" "banana" "cherry")
echo ${fruits[0]}
echo ${fruits[@]}        # All elements
echo ${#fruits[@]}       # Number of elements

# Positional parameters
echo "Script name: $0"
echo "First arg: $1"
echo "All args: $@"
echo "Number of args: $#"

# Exit codes
exit 0                   # Success
exit 1                   # General error
exit 2                   # Misuse of shell builtins

# Debugging
#!/bin/bash -x           # Debug mode
set -x                   # Enable debugging
set +x                   # Disable debugging

```

## **Practical Exercises**

### **Daily Tasks Script**

```bash

#!/bin/bash
# system_report.sh

echo "=== System Report ==="
echo "Date: $(date)"
echo "Uptime: $(uptime -p)"
echo "Memory: $(free -h | awk '/Mem:/ {print $3 "/" $2}')"
echo "Disk: $(df -h / | awk 'NR==2 {print $3 "/" $2}')"
echo "Top processes:"
ps aux --sort=-%mem | head -5

```

### **Backup Script**

```bash

#!/bin/bash
# backup.sh

BACKUP_DIR="/backups"
SOURCE_DIR="/home/user/documents"
DATE=$(date +%Y%m%d_%H%M%S)

if [ ! -d "$BACKUP_DIR" ]; then
    mkdir -p "$BACKUP_DIR"
fi

tar -czf "$BACKUP_DIR/backup_$DATE.tar.gz" "$SOURCE_DIR"

if [ $? -eq 0 ]; then
    echo "Backup completed: backup_$DATE.tar.gz"
else
    echo "Backup failed!"
    exit 1
fi

# Cleanup old backups (keep last 7 days)
find "$BACKUP_DIR" -name "backup_*.tar.gz" -mtime +7 -delete
```

### **User Management Script**

```bash

#!/bin/bash
# create_users.sh

if [ "$(id -u)" -ne 0 ]; then
    echo "Please run as root"
    exit 1
fi

while read -r username; do
    if id "$username" &>/dev/null; then
        echo "User $username already exists"
    else
        useradd -m -s /bin/bash "$username"
        echo "User $username created"
        echo "$username:default123" | chpasswd
        echo "Default password set for $username"
    fi
done < users.txt

```