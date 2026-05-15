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

### uptime — Load Averages

```bash
uptime
```

Output: `up 3 days, 2:14,  2 users,  load average: 1.23, 0.87, 0.65`

The three load average numbers cover the last **1, 5, and 15 minutes**. They count the number of processes either running or waiting for CPU/disk at any moment.

- Load = number of CPU cores → fully utilized
- Load > cores → overloaded (queue building up)
- Falling load (1m < 15m) → problem is recovering; rising (1m > 15m) → getting worse
- Use `nproc` to check how many cores you have

### dmesg — Kernel Ring Buffer

```bash
dmesg -T | tail          # last kernel messages with human-readable timestamps
dmesg -T | grep -i error # filter for errors
dmesg -T | grep -i oom   # Out-Of-Memory killer events
dmesg --level=err,warn   # show only errors and warnings
```

Useful for spotting hardware errors, OOM kills, driver issues, and filesystem errors at boot or runtime.

### vmstat — Virtual Memory Statistics

```bash
vmstat 1            # update every 1 second (Ctrl+C to stop)
vmstat 1 5          # update every 1 sec, 5 times
vmstat -s           # summary table (memory, swap, I/O totals)
vmstat -s -SM       # summary in megabytes
vmstat -d           # disk statistics
vmstat -p <part>    # statistics for a specific partition
```

Key columns:
- `r` — run queue: processes waiting for CPU (> CPU count = saturation)
- `b` — blocked: processes in uninterruptible sleep (waiting for I/O)
- `si`/`so` — swap in / swap out (any non-zero = memory pressure)
- `bi`/`bo` — blocks read/written per second
- `cs` — context switches per second
- `us`/`sy`/`id`/`wa` — CPU: user / system / idle / iowait %

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

### df — Disk Space Usage

```bash
df -h              # all filesystems, human-readable sizes
df -h /path        # usage for the filesystem containing /path
df -i              # inode usage (inode exhaustion can fill a "full" disk with space free)
df -Th             # include filesystem type in output
```

Watch for filesystems above ~85% — some filesystems reserve 5% for root, so 95% reported = truly full for normal users.

### sysstat Package — iostat, mpstat, pidstat, sar

Install: `apt install sysstat` / `dnf install sysstat`

#### iostat — CPU and Disk I/O Statistics

```bash
iostat                  # basic CPU and disk stats since boot
iostat -x               # extended stats (utilization, queue depth, await, etc.)
iostat -xz 1            # extended + hide idle devices, update every 1 sec
iostat -d               # disk stats only
iostat -c               # CPU stats only
iostat -k 1 3           # in kilobytes, every 1 sec, 3 times
iostat -d <disk_name>   # stats for a specific disk (e.g., sda)
```

- `-z` skips devices with zero activity — useful on systems with many disks so you only see the busy ones.

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
sar              # CPU usage
sar -r           # memory usage
sar -b           # I/O transfer rates (reads/writes per second)
sar -b -d 10 1   # I/O rates + per-disk stats, 10 sec interval, 1 sample
sar -S           # swap usage
sar -d           # per-disk activity (device-level)
sar -n DEV 1     # network interface stats (rx/tx packets, KB/s)
sar -n EDEV 1    # network errors per interface
sar -n DEV,EDEV 1 # both interface stats and errors together
sar -n TCP 1     # TCP connection rates (active, passive opens)
sar -n ETCP 1    # TCP errors (retransmits, failed attempts)
sar -n TCP,ETCP 1 # TCP stats + errors combined (key for network triage)
sar -q           # run queue and load average

sar -f /var/log/sa/sa<DD>    # read a specific day's historical data file
sar -s 10:00:00 -e 12:00:00  # filter output to a time range
```

Key `sar -n TCP` columns:
- `active/s` — new outbound TCP connections per second (your app connecting out)
- `passive/s` — new inbound TCP connections per second (your app accepting)
- `retrans/s` — retransmits per second (packet loss / congestion signal)

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

### netstat — Network Connections and Stats

```bash
netstat -tulnp          # TCP/UDP listening ports with process names (no DNS resolve)
netstat -s              # cumulative protocol statistics (TCP errors, UDP drops, etc.)
netstat -rnv            # routing table verbose (check for inefficient/unexpected routes)
netstat -anp | grep <port>  # connections on a specific port
```

- `-t` TCP, `-u` UDP, `-l` listening, `-n` no DNS, `-p` show process

`netstat` is deprecated on modern systems — `ss` is the replacement and is faster:
```bash
ss -tulnp               # same as netstat -tulnp but faster
ss -s                   # summary: counts per state (ESTABLISHED, TIME-WAIT, etc.)
ss -tp state ESTABLISHED # all established TCP connections with process
```

### smartctl — Disk Health (SMART)

```bash
smartctl -l error /dev/sda          # error log (read/write failures)
smartctl -a /dev/sda                # full SMART report (health, attributes, errors)
smartctl -H /dev/sda                # quick health check: PASSED or FAILED
smartctl -t short /dev/sda          # run a short self-test (~2 min)
```

Install: `apt install smartmontools` / `dnf install smartmontools`

Key SMART attributes to watch:
- `Reallocated_Sector_Ct` — sectors remapped due to errors (non-zero = physical damage)
- `Pending_Sector_Count` — sectors waiting to be remapped (imminent failure risk)
- `Uncorrectable_Sector` — unrecoverable read errors

Error log path when available directly: `/sys/devices/.../ioerr_cnt`

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

---

## 6. Advanced Profiling: perf and eBPF/BCC

### perf stat — Hardware Performance Counters

```bash
perf stat -a -- sleep 10        # system-wide counters for 10 seconds
perf stat -p <pid> -- sleep 5   # counters for a specific process
perf stat -a -d -- sleep 10     # detailed: L1/LLC cache, branch misses
```

Key metrics:
- **IPC** (Instructions Per Cycle) — CPU efficiency; < 1.0 suggests pipeline stalls or cache misses
- **LLC-load-misses** — Last Level Cache misses; high rate means data isn't fitting in cache (memory bandwidth bound)
- **branch-misses** — mispredicted branches; indicates speculative execution overhead

### perf record / report — CPU Flame Graph Data

```bash
perf record -ag -- sleep 10     # record call graphs system-wide for 10 sec
perf report                     # interactive TUI to explore the profile
perf script | flamegraph.pl > out.svg  # generate flame graph (with FlameGraph scripts)
```

A CPU flame graph shows where time is actually being spent — each horizontal bar is a stack frame, width = time proportion. Top of the flame = where the CPU was when sampled.

### eBPF / BCC Tools

BCC tools use eBPF to instrument the kernel with near-zero overhead. Install: `apt install bpfcc-tools` / `dnf install bcc-tools`

**Filesystem I/O latency:**
```bash
ext4slower 10        # ext4 operations slower than 10ms (also xfsslower, zfsslower, btrfsslower)
ext4dist 1           # histogram of ext4 operation latency distribution, 1 sec intervals
```

**Block I/O latency:**
```bash
bioslower 10         # block I/O requests slower than 10ms
biolatency 1         # histogram of block I/O latency; spot bimodal distributions
biosnoop             # per-I/O trace: process, disk, latency, size
```

**TCP / network:**
```bash
tcpretrans           # trace every TCP retransmit with remote address and TCP state
tcpconnect           # trace outbound TCP connections (spot unexpected connections)
tcpaccept            # trace inbound TCP connections (unexpected workload detection)
tcptop 1             # live top-like view of TCP throughput by connection
```

**CPU / scheduling:**
```bash
runqlat 1            # histogram of CPU run-queue latency (time waiting to run)
offcputime -p <pid>  # time process spent off-CPU and why (I/O, locks, etc.)
```

eBPF tools are safe on production — they add instrumentation points but do not modify kernel code and have built-in safety verification.

---

## 7. Performance Checklists

### Linux Perf Analysis in 60 Seconds

Quick triage commands to run in order on an unknown performance issue:

```bash
uptime                  # load averages — is the system overloaded?
dmesg -T | tail         # any recent kernel errors or OOM kills?
vmstat 1                # overall: CPU idle, run queue, swap, I/O
mpstat -P ALL 1         # CPU balance — is load spread or on one core?
pidstat 1               # which process is consuming CPU?
iostat -xz 1            # any disk I/O? utilization? await?
free -m                 # memory and swap usage
sar -n DEV 1            # network throughput per interface
sar -n TCP,ETCP 1       # TCP connection rates and retransmits
top                     # cross-check, look for anything missed
```

### Disk Checklist

```bash
iostat -xz 1            # any disk I/O at all? if nothing, stop looking here
vmstat 1                # swapping (si/so)? high sys time?
df -h                   # are filesystems nearly full?
ext4slower 10           # slow filesystem operations? (xfsslower, zfsslower for other fs)
bioslower 10            # if fs is slow, are the disks slow?
ext4dist 1              # check latency distribution and operation rates
biolatency 1            # block I/O latency histogram — any outliers or bimodal shape?
smartctl -l error /dev/sda   # disk error log (if available)
cat /sys/devices/.../ioerr_cnt  # hardware I/O error counter (if available)
```

### Network Checklist

```bash
sar -n DEV,EDEV 1       # at interface limits? any errors? (or: nicstat)
sar -n TCP,ETCP 1       # connection load and retransmit rate
cat /etc/resolv.conf    # verify DNS config (it's always DNS)
mpstat -P ALL 1         # high kernel time? single hot CPU (soft-IRQ for NIC)?
tcpretrans              # what are the retransmits? TCP state at time of retransmit?
tcpconnect              # connecting to anything unexpected?
tcpaccept               # unexpected inbound workload?
netstat -rnv            # any inefficient or unexpected routes?
netstat -s              # protocol-level stats: drops, errors, retransmits
# check firewall rules  # anything blocking or rate-limiting traffic?
```

### CPU Checklist

```bash
uptime                  # load averages trend (1m vs 15m)
vmstat 1                # system-wide utilization, run queue depth
mpstat -P ALL 1         # per-core balance — is one core hot (single-threaded bottleneck)?
pidstat 1               # per-process CPU usage
perf stat -a -- sleep 10  # IPC and LLC hit ratio (are we memory-bound?)
# CPU flame graph       # profile with: perf record -ag -- sleep 30 && perf report
# subsecond offset heat map  # reveals scheduler gaps or lock contention
```
