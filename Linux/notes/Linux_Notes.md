Below are **mid-level interview notes for Linux (DevOps / Platform Engineering)** focused on **real operational knowledge**, troubleshooting ability, and production thinking.

---

# Linux for DevOps / Platform Engineering — Mid-Level Interview Notes

---

# 1. Core Concepts (Mid-Level Focus)

## 1. Linux Architecture

Linux follows a **modular architecture** consisting of:

```
User Applications
↓
System Libraries
↓
Shell
↓
Kernel
↓
Hardware
```

### Kernel

The **kernel is the core of the OS** responsible for:

* Process scheduling
* Memory management
* Device drivers
* Networking
* File systems

Why it matters for DevOps:

* Containers use kernel features (namespaces, cgroups)
* Performance tuning often involves kernel parameters

---

## 2. Linux Kernel vs User Space

### Kernel Space

Runs privileged operations.

Examples:

* Process scheduling
* Memory allocation
* Hardware interaction

### User Space

Where applications run.

Examples:

* bash
* nginx
* docker
* kubectl

Key concept:
User applications interact with kernel via **system calls**.

---

## 3. Process Management

A **process** is a running instance of a program.

Important attributes:

* PID (Process ID)
* PPID (Parent PID)
* UID (User ID)
* State

Common process states:

| State | Meaning               |
| ----- | --------------------- |
| R     | Running               |
| S     | Sleeping              |
| D     | Uninterruptible sleep |
| T     | Stopped               |
| Z     | Zombie                |

DevOps relevance:

* debugging stuck services
* CPU usage analysis
* identifying zombie processes

---

## 4. File System Hierarchy

Linux follows **FHS (Filesystem Hierarchy Standard)**.

Important directories:

| Directory | Purpose               |
| --------- | --------------------- |
| /         | root                  |
| /bin      | essential binaries    |
| /etc      | configuration files   |
| /var      | logs, variable data   |
| /home     | user home directories |
| /tmp      | temporary files       |
| /usr      | user binaries         |
| /opt      | third-party software  |

Example:

```
/var/log/nginx/access.log
/etc/nginx/nginx.conf
```

---

## 5. Permissions and Ownership

Linux security is based on:

```
Owner
Group
Others
```

Permission types:

| Permission | Meaning |
| ---------- | ------- |
| r          | read    |
| w          | write   |
| x          | execute |

Example:

```
-rwxr-xr--
```

Interpretation:

Owner: rwx
Group: r-x
Others: r--

Commands:

```
chmod
chown
chgrp
```

---

## 6. Systemd and Services

Modern Linux distributions use **systemd**.

Responsibilities:

* service management
* logging
* boot process
* resource control

Commands:

```
systemctl start nginx
systemctl stop nginx
systemctl restart nginx
systemctl status nginx
```

DevOps relevance:

* service automation
* startup scripts
* dependency ordering

---

## 7. Networking in Linux

Key networking components:

* network interfaces
* routing table
* firewall rules
* ports and sockets

Common tools:

```
ip
ss
netstat
ping
curl
traceroute
```

Example:

```
ss -tulnp
```

Shows listening ports.

---

# 2. Key Terminology & Definitions

### Daemon

Background service process.

Example:

```
sshd
nginx
docker
```

---

### Fork

Creating a new process from an existing one.

---

### Zombie Process

A process that has finished execution but still has an entry in the process table.

---

### Load Average

Average number of processes waiting for CPU.

Example:

```
load average: 1.2 0.8 0.5
```

---

### Swap

Disk used as **virtual memory when RAM is full**.

---

### Inode

Metadata of a file.

Contains:

* file size
* owner
* permissions
* timestamps

---

### Soft Link vs Hard Link

Hard Link:

* same inode
* cannot cross filesystems

Soft Link:

* pointer to file
* like a shortcut

---

# 3. Tools, Frameworks & Technologies

## System Monitoring

### top

Real-time process monitoring.

### htop

Better UI version of top.

### vmstat

Memory statistics.

### iostat

Disk IO monitoring.

### sar

Historical system metrics.

---

## File Operations

```
ls
cp
mv
rm
find
locate
```

Example:

```
find /var/log -name "*.log"
```

---

## Text Processing Tools

Critical for DevOps pipelines.

### grep

Search text.

```
grep "error" logs.txt
```

### awk

Data processing.

### sed

Stream editing.

Example:

```
sed 's/error/warning/g'
```

---

## Disk Usage

```
df -h
du -sh
lsblk
mount
```

---

# 4. Comprehensive Interview Questions & Answers

---

# Q1: How does Linux handle processes?

### Answer

Processes are managed by the **Linux scheduler**.

Key points:

* each process has PID
* parent-child relationship
* scheduler allocates CPU time slices

Process lifecycle:

```
New → Ready → Running → Waiting → Terminated
```

Example commands:

```
ps aux
top
htop
```

Mid-level insight:

Use `ps aux --sort=-%cpu` to find CPU-heavy processes.

---

# Q2: How would you troubleshoot high CPU usage?

Step-by-step production approach:

### Step 1: Identify process

```
top
```

or

```
ps aux --sort=-%cpu
```

### Step 2: Inspect process

```
ps -fp <PID>
```

### Step 3: Check logs

```
/var/log
journalctl
```

### Step 4: Check thread usage

```
top -H -p <PID>
```

### Step 5: Kill or restart if necessary

```
kill -9 PID
systemctl restart service
```

---

# Q3: Difference between soft link and hard link?

Hard Link

* points to same inode
* survives original deletion

Soft Link

* pointer to path
* breaks if original deleted

Example:

```
ln file1 file2
ln -s file1 symlink
```

---

# Q4: What happens when disk becomes full?

Possible causes:

* large logs
* docker images
* temporary files
* backups

Troubleshooting:

```
df -h
du -sh *
```

Find largest directories.

```
du -h / | sort -rh | head
```

---

# Q5: How does Linux boot process work?

Stages:

1. BIOS/UEFI
2. Bootloader (GRUB)
3. Kernel loading
4. Init system (systemd)
5. Services start

---

# Q6: What is Load Average?

Represents number of processes waiting for CPU.

Example:

```
1.5 on 2 CPU cores = healthy
5.0 on 2 CPU cores = overloaded
```

---

# 5. Code Snippets / Pseudocode

Example script for monitoring disk usage.

```bash
#!/bin/bash

usage=$(df / | tail -1 | awk '{print $5}' | sed 's/%//')

if [ "$usage" -gt 80 ]; then
    echo "Disk usage critical"
fi
```

---

# 6. Practical Real-World Scenarios

## Scenario: Production server becomes slow

Investigation:

1. CPU

```
top
```

2. Memory

```
free -m
```

3. Disk IO

```
iostat
```

4. Network

```
iftop
```

5. Logs

```
journalctl
```

---

## Scenario: Container failing in Kubernetes

Possible Linux causes:

* disk full
* permissions issue
* file descriptor limit
* memory OOM

Commands:

```
dmesg
journalctl
ulimit -n
```

---

# 7. Performance & Optimization

### CPU

Check:

```
mpstat
```

Tune:

```
nice
renice
```

---

### Memory

Check:

```
free -m
vmstat
```

Look for:

* swap usage
* OOM killer

---

### Disk IO

Check:

```
iostat -x
```

High:

```
%util
await
```

---

# 8. Debugging & Troubleshooting

Example: Service not starting.

Steps:

```
systemctl status nginx
journalctl -xe
```

Check:

* port conflict
* permissions
* missing dependencies

Check port:

```
ss -tulnp
```

---

# 9. Security Considerations

### Principle of Least Privilege

Do not run services as root.

---

### SSH Hardening

Disable root login.

```
PermitRootLogin no
```

---

### Firewall

```
ufw
iptables
```

Example:

```
ufw allow 22
```

---

# 10. Follow-Up Interview Questions

Interviewers probe deeper with:

* What happens when inode limit is reached?
* Difference between process and thread?
* How does Linux memory management work?
* How does cgroups limit containers?
* What happens when swap is exhausted?

---

# 11. Common Mistakes & Red Flags

### Red Flag 1

Candidate only knows commands, not concepts.

Example bad answer:

"Use top."

Better answer:

"Use top to identify process, then analyze CPU threads."

---

### Red Flag 2

Does not understand load average.

---

### Red Flag 3

Does not know difference between:

```
df
du
```

---

# 12. Behavioral Signals

Strong candidate signals:

* systematic debugging
* production thinking
* understands logs first approach
* cautious with kill -9

---

# 13. Evaluation Guide

### Junior

* knows commands
* basic Linux navigation

### Mid-Level

* troubleshooting production issues
* understands kernel resources
* performance analysis

### Senior

* kernel tuning
* system architecture
* capacity planning

---

# 14. Flashcards (Quick Revision)

### What is inode?

Metadata structure describing file.

---

### What command checks disk usage?

```
df -h
```

---

### Command to find process by port

```
lsof -i :8080
```

---

### Command to see listening ports

```
ss -tulnp
```

---

### Command to view logs

```
journalctl
```
---