# Linux Performance

---

## 1. Architecture Concepts

### Context Switch

A context switch happens when the CPU stops executing one process and starts executing another. The kernel saves the current process state (registers, program counter, etc.) and loads the saved state of the next process.

- Too many context switches can degrade performance due to cache invalidation and overhead.
- To see the number of context switches for a process:
  ```
  grep ctxt /proc/<pid>/status
  ```
  - `voluntary_ctxt_switches`: process gave up the CPU willingly (e.g., waiting for I/O)
  - `nonvoluntary_ctxt_switches`: kernel preempted the process (e.g., time slice expired)

- System-wide context switches per second:
  ```
  vmstat 1        # cs column
  ```

### Run Queue

The run queue holds all processes in a **runnable** state waiting to be scheduled on a CPU.

- A run queue length greater than the number of CPU cores indicates CPU saturation.
- Check run queue with `vmstat`:
  ```
  vmstat 1
  ```
  - `r` column: number of processes waiting for CPU time
  - `b` column: number of processes in uninterruptible sleep (waiting for I/O)

### I/O Wait

I/O wait is the percentage of time the CPU is idle **because** it is waiting for an I/O operation to complete. High iowait suggests a storage bottleneck.

- `vmstat` I/O columns:
  - `bi` → blocks read per second (block input)
  - `bo` → blocks written per second (block output)

- `iostat` terminology:
  - `tps` → total transactions per second = `rtps` (read tps) + `wtps` (write tps)
  - `%iowait` → percent of CPU time spent waiting for I/O

---

## 2. Kernel: /proc, Tunables, Modules, tuned

### /proc Filesystem

`/proc` is a **virtual filesystem** — it is not stored on disk and is recreated at every boot. It exposes kernel data structures as files.

- `/proc/sys/` — tunable kernel parameters (read/write)
- `/proc/meminfo` — memory statistics (read-only)
- `/proc/cpuinfo` — CPU details (read-only)
- `/proc/<pid>/status` — per-process info (read-only)
- `/proc/<pid>/fd/` — open file descriptors for a process

### Modifying Kernel Tunables

**Method 1 — Edit directly in /proc/sys** (not persistent, lost on reboot):
```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

**Method 2 — sysctl command:**
```bash
sysctl -a                          # list all parameters
sysctl -n <parameter>              # show value of a specific parameter
sysctl -w <parameter=value>        # set value (not persistent)
sysctl net.ipv4.ip_forward=1
```

**Method 3 — Persistent via /etc/sysctl.d** (survives reboot):
1. Create a `.conf` file in `/etc/sysctl.d/`:
   ```
   # /etc/sysctl.d/99-custom.conf
   net.ipv4.ip_forward = 1
   vm.swappiness = 10
   ```
2. Apply immediately without rebooting:
   ```bash
   sysctl -p /etc/sysctl.d/99-custom.conf
   ```

### Linux Kernel Modules

Modules are pieces of code that can be loaded/unloaded into the kernel on demand, without rebooting.

```bash
lsmod                        # list all currently loaded modules
modinfo <module_name>        # show info (version, parameters, dependencies)
modprobe <module_name>       # load a module (also loads dependencies)
modprobe -r <module_name>    # remove a module
```

- Config for persistent module loading: `/etc/modules-load.d/*.conf`
- Module parameters can be set in `/etc/modprobe.d/*.conf`

### tuned

`tuned` is a daemon that dynamically adjusts kernel parameters based on a selected profile for your workload.

```bash
tuned-adm list                    # list available profiles
tuned-adm active                  # show current active profile
tuned-adm profile <profile_name>  # switch to a profile
tuned-adm recommend               # recommend a profile for this system
```

Common profiles: `balanced`, `throughput-performance`, `latency-performance`, `powersave`, `virtual-guest`, `network-throughput`

---

## 3. Monitoring Tools

### ps — Process Snapshot

```bash
ps -fU <username>                               # processes run by a specific user
ps -p $(pidof <cmd>)                            # processes related to a command
ps ax --format pid,%mem,cmd --sort -%mem        # sort by memory usage descending
ps --forest -C <cmd> -o pid,ppid,rss,%cpu       # show process hierarchy (tree view)
```

- `rss` — Resident Set Size: actual physical memory used (in KB)
- `%cpu` — cumulative CPU usage
- `--forest` — ASCII art tree showing parent/child relationships

### top — Live Process Monitor

```bash
top -n 1        # run once and exit (useful in scripts)
top -u <user>   # show only processes for a specific user
top -p <pid>    # monitor a specific PID
```

Interactive shortcuts inside top:
- `Shift+M` — sort by memory usage
- `Shift+T` — sort by cumulative CPU time
- `Shift+N` — sort by PID
- `Shift+P` — sort by CPU usage (default)
- `k` — kill a process (prompts for PID)
- `r` — renice a process
- `1` — toggle per-CPU core view
- `q` — quit

### free — Memory Usage

```bash
free -h             # human-readable (MB/GB)
free -s <sec>       # refresh output every <sec> seconds
free -s 2 -c 5      # refresh every 2 sec, 5 times total
```

Key columns:
- `total` — total installed RAM
- `used` — memory in use
- `free` — completely unused memory
- `buff/cache` — memory used for disk buffers and cache (reclaimable)
- `available` — memory available for new processes (free + reclaimable cache)

### lsblk — Block Devices

```bash
lsblk              # list block devices in tree format
lsblk -f           # full format: filesystem type, label, UUID, mount point
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINT   # custom columns
```

### sysstat Package — iostat, mpstat, pidstat, sar

Install: `apt install sysstat` / `dnf install sysstat`

#### iostat — CPU and Disk I/O Statistics

```bash
iostat                  # basic CPU and disk stats since boot
iostat -x               # extended stats (utilization, queue depth, await, etc.)
iostat -d               # disk stats only
iostat -c               # CPU stats only
iostat -k 1 3           # in kilobytes, every 1 sec, 3 times
iostat -d <disk_name>   # stats for a specific disk (e.g., sda)
```

Key extended columns:
- `%util` — percentage of time disk was busy (100% = saturated)
- `await` — average time (ms) for I/O requests (queue + service time)
- `r_await` / `w_await` — read/write await time
- `aqu-sz` — average queue size

#### mpstat — CPU Per-Core Statistics

```bash
mpstat              # average across all cores
mpstat -P ALL       # stats per individual core
mpstat -P <core>    # stats for a specific core number
mpstat 1 5          # update every 1 sec, 5 times
```

Key columns:
- `%usr` — user space CPU usage
- `%sys` — kernel space CPU usage
- `%iowait` — waiting for I/O
- `%idle` — idle time

#### pidstat — Per-Process Statistics

```bash
pidstat                     # stats for active processes (default: CPU)
pidstat -p ALL              # stats for ALL processes
pidstat -C <cmd> -l         # stats for processes matching command name; -l shows full command
pidstat -r                  # memory usage stats per process
pidstat -d                  # disk I/O stats per process
pidstat 1 5                 # update every 1 sec, 5 times
```

#### sar — System Activity Reporter (historical data)

`sar` collects and reports system activity. Data is stored in `/var/log/sa/` (or `/var/log/sysstat/`).

```bash
sar           # CPU usage
sar -r        # memory usage
sar -b        # I/O and transfer rates
sar -S        # swap usage
sar -d        # disk activity
sar -n DEV    # network interface statistics
sar -q        # run queue and load average

sar -f /var/log/sa/sa<DD>   # read a specific day's data file
sar -s 10:00:00 -e 12:00:00 # filter by time range
```

---

## 4. File and Process Tools

### lsof — List Open Files

Everything in Linux is a file (sockets, pipes, devices). `lsof` shows which processes have files open.

```bash
lsof -u <user>          # open files for a specific user
lsof +D /path/to/dir    # all open files inside a directory (recursive)
lsof -c <process_name>  # open files for processes with this name (e.g., sshd, nginx)
lsof -p <pid>           # open files for a specific PID
lsof -i                 # all network connections
lsof -i :80             # processes using port 80
lsof -i tcp             # all TCP connections
lsof /path/to/file      # which process has this file open
```

Useful for finding "device busy" errors before unmounting:
```bash
lsof +D /mnt/usb
```

### fuser — Find Processes Using a File or Socket

```bash
fuser /path/to/file         # show PIDs using the file
fuser -cu /path             # show PIDs and usernames (-c), summary (-u)
fuser -k /path/to/file      # kill all processes using the file
fuser -v /path              # verbose output
fuser -n tcp 80             # processes using TCP port 80
```

### tcpdump — Packet Capture

```bash
tcpdump -i any                          # capture on all interfaces
tcpdump -i eth0                         # capture on a specific interface
tcpdump -w <filename.pcap> -i any       # write raw packets to file
tcpdump -r <filename.pcap>              # read/replay from file

tcpdump host 192.168.1.1                # filter by host
tcpdump port 443                        # filter by port
tcpdump 'tcp and port 80'               # combine filters
tcpdump -n                              # don't resolve hostnames (faster)
tcpdump -v / -vv / -vvv                 # increasing verbosity

# Quick cheatsheet (requires curl)
curl cheat.sh/tcpdump
```

---

## 5. Resource Limits (ulimit)

Linux enforces per-process resource limits to prevent any single process from exhausting system resources.

```bash
ulimit -a           # show all current limits for the shell session
ulimit -n           # show max open file descriptors
ulimit -u           # show max user processes
ulimit -n 65536     # set max open file descriptors (soft, session only)
```

### Soft vs Hard Limits

- **Soft limit** — the current enforced limit; a user/process can raise it up to the hard limit with `ulimit`.
- **Hard limit** — the ceiling; only root can raise it.

### Persistent Limits — /etc/security/limits.conf

```
# /etc/security/limits.conf
# <domain>  <type>  <item>   <value>
*           soft    nofile   65536
*           hard    nofile   100000
nginx       soft    nproc    4096
@devteam    hard    memlock  unlimited
```

- `*` — applies to all users
- `@groupname` — applies to a group
- Items: `nofile` (open files), `nproc` (processes), `memlock`, `stack`, `core`, `cpu` (CPU seconds)

For drop-in files: `/etc/security/limits.d/<name>.conf` (same syntax, per user/group)

### Systemd Service Limits

For services managed by systemd, `limits.conf` does NOT apply by default. Use:

```bash
systemctl set-property <service> LimitNOFILE=65536    # persistent
systemctl set-property --runtime <service> LimitNOFILE=65536  # until next reboot
```

Or add directly to the unit file under `[Service]`:
```ini
[Service]
LimitNOFILE=65536
LimitNPROC=4096
```

Check effective limits of a running process:
```bash
cat /proc/<pid>/limits
```
