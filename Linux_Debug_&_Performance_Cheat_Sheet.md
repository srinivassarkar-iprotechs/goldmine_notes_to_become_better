# The Ultimate Linux Debug & Performance Cheat Sheet

> Because Linux doesn't come with a help button.

## Part 1: 7-Step Debugging Cheat Sheet

### 1. Verify the Obvious
```bash
ps aux | grep process_name
pgrep -fl process_name
dmesg -T | tail -50
```
- Is it running?
- Check process name.
- Review kernel logs (segfaults/OOM).

### 2. Check Logs
```bash
journalctl -xe --no-pager -n 50
tail -f /var/log/syslog
```
- Recent system errors
- Live log feed

### 3. Resource Check
```bash
top -o %CPU
top -o %MEM
dmesg | grep -i oom
```
- CPU hogs
- Memory overuse
- OOM killer activity

### 4. Storage Issues
```bash
df -h
iostat -xm 1
dmesg | grep -i error
```
- Disk usage
- I/O bottlenecks
- Filesystem errors

### 5. File Locks
```bash
lsof -p <PID>
lsof | grep -iE "deleted|locked"
```
- Open files
- Deleted/locked files

### 6. External Kills
```bash
journalctl -u process_name --no-pager -n 50
lastcomm | grep process_name
```
- Service kill logs
- Kill history

### 7. Real-Time Debugging
```bash
strace -p <PID>
gdb -p <PID>
```
- Trace syscalls/signals
- Attach debugger

> Follow the steps. Stay calm. The problem always leaves a trail.

---

## Part 2: Linux Performance Analysis Cheat Sheet

### System Load & Uptime
```bash
uptime
```
Shows current time, uptime, and load averages (1, 5, 15 min).

### Kernel Logs
```bash
dmesg | tail
```
Shows latest kernel messages (hardware, driver, crash info).

### CPU, Memory & I/O Overview
```bash
vmstat 1
```
Displays processes, memory, swap, I/O, and CPU usage.

### CPU Per-Core Usage
```bash
mpstat -P ALL 1
```
Shows CPU usage statistics for every core.

### Per-Process Resource Usage
```bash
pidstat 1
```
Displays CPU, memory, and I/O usage per process.

### Detailed Disk I/O
```bash
iostat -xz 1
```
Shows extended disk statistics (IOPS, latency, utilization).

### Memory Usage
```bash
free -m
```
Displays RAM and swap usage.

### Network Usage
```bash
sar -n DEV 1
```
Shows interface RX/TX packets and bandwidth.

### TCP Connections & Errors
```bash
sar -n TCP,ETCP 1
```
Displays active/passive connections, retransmits, resets, and errors.

### Interactive Monitor
```bash
top
```
Real-time CPU, memory, process, and load monitoring.

---

## Principles

- Be curious.
- Read logs.
- Trust data.
- Repeat.

**Measure. Analyze. Optimize.**
