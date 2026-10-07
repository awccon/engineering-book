# Troubleshooting Linux

Something is wrong with a server. The site is slow, or down, or a deployment failed, or an
alert says "high load." You SSH in (or open a container shell) and face a blinking cursor.
What do you type first?

This chapter is a playbook: a systematic method for investigating a Linux machine under
pressure, organized by resource (CPU, memory, disk, network, processes, logs), with the
commands that answer each question quickly. It's the Linux counterpart to Book III,
Chapter 10's API diagnosis method.

---

## 1. The problem: many symptoms, few root causes

Symptoms are vague: "slow," "hanging," "down," "errors." Underneath, most server problems
are one of a small number of things:

- a resource is **saturated**: CPU, memory, disk space, disk I/O, network, file descriptors,
  connections;
- a **process** is dead, stuck, restarting or misconfigured;
- a **dependency** (DNS, database, another service) is failing;
- a **change** (deployment, configuration, certificate, OS update) broke something.

The method is to check each quickly, narrow down, then dig.

---

## 2. The mental model: the first 60 seconds

A widely used checklist (popularized by Brendan Gregg at Netflix): ten commands that give a
broad picture of a Linux system in about a minute.

```bash
uptime                      # 1. load averages: trend over 1, 5, 15 minutes
dmesg -T | tail -20         # 2. kernel messages: OOM kills, disk errors, network issues
vmstat 1 5                  # 3. CPU, memory, swap, run queue, I/O wait, per second
mpstat -P ALL 1 3           # 4. per-CPU usage: one core pegged?
pidstat 1 3                 # 5. which processes use CPU
iostat -xz 1 3              # 6. disk utilization, latency, queue depth
free -h                     # 7. memory and swap
sar -n DEV 1 3              # 8. network throughput per interface
sar -n TCP,ETCP 1 3         # 9. TCP connections, retransmits
top                         # 10. overview; then press 1 (per CPU), M (sort by memory), P (by CPU)
```

(`mpstat`, `pidstat`, `iostat` and `sar` come from the `sysstat` package; install it on
every server before you need it.)

This is the **USE method** in practice (Book III, Chapter 10): for each resource, check
**Utilization**, **Saturation** and **Errors**.

> **🧱 Durable:** Start broad and cheap, then narrow. Ten quick commands that rule things
> out beat an hour of deep investigation in the wrong place. And before any of it, ask:
> *what changed?*

---

## 3. CPU

### Load average

```bash
$ uptime
 14:02:11 up 41 days,  load average: 7.82, 6.95, 3.10
```

Load average is the average number of processes **running or waiting to run** (on Linux,
also those waiting on uninterruptible I/O). Compare it to the number of CPUs (`nproc`): a
load of 7.8 on 2 CPUs means heavy contention; on 16 CPUs, it's fine. The three numbers show
the trend: here, load has risen recently.

### Who is using CPU, and how?

```bash
top                         # %CPU per process; "us" user, "sy" system, "wa" I/O wait, "st" steal
pidstat -u 1 5              # per-process CPU over time
```

| `top` field | High means |
|---|---|
| `us` (user) | Application code is busy: your app, a runaway process |
| `sy` (system) | Kernel work: many syscalls, context switches, networking |
| `wa` (I/O wait) | CPUs idle waiting for disk: an I/O problem, not a CPU problem |
| `st` (steal) | The hypervisor gave your vCPU's time to another VM: noisy neighbor or burstable instance out of credits |

For a .NET process using too much CPU, go deeper with `dotnet-trace` or `dotnet-counters`
(Book I, Chapter 11): what code is hot, is the GC busy?

> **⚠️ What can go wrong:** Cloud "burstable" instance types (B-series on Azure, T-series on
> AWS) run fast until they exhaust CPU credits, then are throttled heavily. Sudden,
> sustained slowness with high `st` or low CPU at full load is a classic sign.

---

## 4. Memory

```bash
$ free -h
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       3.1Gi       112Mi        12Mi       600Mi       480Mi
Swap:             0B          0B          0B
```

- **`available`** is the number that matters: memory that can be given to applications
  (free plus reclaimable cache). Low `free` is normal; Linux uses spare memory for disk cache.
- **Swap** use and swapping activity (`vmstat`'s `si`/`so` columns) mean memory pressure.
  Heavy swapping makes everything slow.

When memory runs out, the kernel's **OOM killer** kills a process:

```bash
dmesg -T | grep -i -E 'killed process|out of memory'
journalctl -k --since today | grep -i oom
```

Which processes use memory:

```bash
ps aux --sort=-rss | head -10           # resident memory (RSS) per process
top -o %MEM
```

For .NET processes: GC heap vs total (`dotnet-counters`), heap contents (`dotnet-gcdump`),
and the leak investigation from Book I, Chapter 11. In containers, the limit is the cgroup's,
not the host's (Chapter 6).

---

## 5. Disk

Two different problems: **space** and **I/O performance**.

### Space

```bash
df -h                                    # free space per filesystem
df -i                                    # free inodes (many small files can exhaust these first)
sudo du -xh / --max-depth=1 | sort -h    # what's using space, one level at a time
sudo du -sh /var/log/* | sort -h
```

Common culprits: logs (app logs written to files without rotation, the journal), Docker
images and build cache (`docker system df`, `docker system prune`), core dumps, old
releases, database WAL growth (Book IV, Chapter 3).

> **⚠️ What can go wrong:** A full disk breaks things in strange ways: databases stop
> accepting writes, applications fail to write temp files, logs stop (so the evidence
> disappears), and package upgrades fail halfway. Also, **deleted files held open by a
> process** still consume space: `sudo lsof +L1` lists them; restarting the process (or
> truncating the file) frees the space.

### I/O performance

```bash
iostat -xz 1 5
```

Look at `%util` (how busy the device is), `r_await`/`w_await` (average latency in ms) and
`aqu-sz` (queue length). High await and queueing mean the disk is a bottleneck. Find which
processes are doing I/O with `iotop` or `pidstat -d 1`. Cloud disks have IOPS and throughput
limits per size/tier; hitting them looks exactly like this.

---

## 6. Network

```bash
ss -s                                    # socket summary: total, TCP states
ss -tlnp                                 # listening sockets and their processes
ss -tnp state established | wc -l        # number of established connections
ss -tn state time-wait | wc -l           # TIME_WAIT: connection churn (Chapter 2)
sar -n DEV 1 5                           # throughput per interface
sar -n ETCP 1 5                          # retransmits: packet loss or congestion
ip -s link                               # interface errors and drops
```

Connectivity checks (Chapter 2's layered method):

```bash
getent hosts api.partner.com             # DNS
nc -vz db.internal 5432                  # TCP reachability
curl -sv https://api.partner.com/health  # HTTP + TLS (certificate errors show here)
mtr -rwc 20 api.partner.com              # path, latency and loss per hop
```

### File descriptors

Every socket and open file uses a **file descriptor**. Processes have limits:

```bash
cat /proc/<pid>/limits | grep 'open files'
ls /proc/<pid>/fd | wc -l                # how many are open now
```

"Too many open files" errors under load mean the limit is too low (raise `LimitNOFILE` in
the systemd unit; Chapter 2) or descriptors are leaking (undisposed connections and streams,
Book I, Chapter 11).

---

## 7. Processes and services

```bash
systemctl --failed                        # failed units
systemctl status beacon-api               # state, recent logs, restarts
journalctl -u beacon-api --since "30 min ago"
ps -eo pid,ppid,user,stat,etime,%cpu,%mem,cmd --sort=-%cpu | head
```

Process states in `ps` (`STAT` column): `R` running, `S` sleeping (normal), `D`
uninterruptible sleep (usually waiting on I/O: many `D` processes point to disk or NFS
problems), `Z` zombie (finished but not reaped by its parent).

What is a process doing right now?

```bash
sudo strace -f -p <pid> -e trace=network,file -tt     # system calls (heavy; use briefly)
sudo lsof -p <pid>                                    # open files and sockets
cat /proc/<pid>/status                                # memory, threads, state
dotnet-stack report -p <pid>                          # managed stacks for .NET (Book III, Ch. 10)
```

---

## 8. Logs

Logs answer "what happened," and they're often the fastest route to the cause:

```bash
journalctl -p err -b                      # all errors since boot
journalctl -u nginx -u beacon-api --since "10 min ago"
journalctl -k                             # kernel messages
tail -f /var/log/nginx/error.log
grep -E ' (5[0-9]{2}) ' /var/log/nginx/access.log | tail     # recent 5xx responses
last -n 20                                # recent logins
journalctl -u ssh --since today | grep -i failed
```

Correlate timestamps across logs: the Nginx 502s started at 13:42:10; what did the app log at
13:42? Did the kernel kill something at 13:42:09?

---

## 9. A worked incident: Beacon is slow on a single server

**Alert (09:20):** p95 latency on `beacon.example.com` above 3 seconds; some 504s.

**What changed?** Last deploy was two days ago. No configuration changes. But a large
customer onboarded this morning and started a bulk import.

**First 60 seconds:**

```text
uptime         load average: 3.95, 3.80, 2.10            (2 vCPUs → saturated recently)
vmstat 1       r=4  ... wa=45  st=0                       (high I/O wait!)
iostat -xz 1   sda: %util 99.8  w_await 180ms  aqu-sz 22  (disk saturated, writes slow)
free -h        available 1.1Gi, no swap                   (memory fine)
df -h          / 71% used                                 (space fine for now)
pidstat -d 1   postgres: kB_wr/s 48000                    (PostgreSQL writing heavily)
```

The CPUs are mostly **waiting on disk**, and PostgreSQL is the writer.

**Narrow down:** in PostgreSQL, `pg_stat_activity` (Book IV, Chapter 5) shows the bulk import
running thousands of single-row `insert`s, each in its own transaction (each commit forces a
WAL flush to disk), while autovacuum works on the growing table. The API's queries wait behind
the saturated disk; the API's thread pool queue grows (`dotnet-counters`), and Nginx times out
at 60 s → 504s.

**Mitigate (09:31):** pause the import job (it's a background job, so the business impact is
a delay, not data loss). Latency recovers within a minute as the disk queue drains.

**Fix:**
- The import uses batched inserts (`COPY` or multi-row inserts in transactions of ~1,000 rows),
  cutting WAL flushes by orders of magnitude (Book IV, Chapter 2).
- Imports run with a concurrency limit and lower priority, off-peak where possible.
- Longer term: the database moves to a managed service with provisioned IOPS (Book IX), and the
  API and database stop sharing one small disk.
- Add a disk latency alert (`w_await` / disk queue), which would have pointed at the cause
  before users noticed.

The method worked: broad checks ruled out CPU, memory and space in a minute, and pointed
straight at disk I/O and its source.

---

## 10. Tools to install before you need them

On every server (or in a debug image):

```bash
sudo apt install -y sysstat htop iotop lsof strace tcpdump dnsutils mtr-tiny jq curl netcat-openbsd
sudo systemctl enable --now sysstat       # records history for `sar` (what happened at 03:00 last night?)
dotnet tool install -g dotnet-counters dotnet-trace dotnet-dump dotnet-gcdump dotnet-stack
```

Historical data is as important as live data: `sar` keeps system history; your monitoring
platform (Book IX, Chapter 6) keeps application metrics. Without history, you can only
diagnose problems that are still happening.

---

## 11. What can go wrong (in the investigation)

- **Restarting first, investigating never**: the problem returns, and the evidence is gone.
  If you must restart to restore service, capture evidence first (a `dotnet-dump`, `ss -s`,
  `top -b -n1`, recent logs).
- **Confusing low "free" memory with a problem** (it's disk cache).
- **Blaming CPU for I/O wait.**
- **Missing deleted-but-open files** when disk space doesn't add up.
- **Heavy tools on a struggling box**: `strace` and full dumps add load; use them briefly and
  deliberately.
- **No history**, so intermittent problems can't be reconstructed.

---

## 12. How an experienced engineer thinks about this

- **What changed?** first, always.
- **Broad, then narrow**: the first-60-seconds checklist, then the resource that stands out.
- **Utilization, saturation, errors** for every resource.
- **Preserve evidence** before restarting.
- **Leave the system better**: an alert, a dashboard, a runbook entry for next time.

---

## 13. Check yourself

**Questions**

1. What does load average measure? How do you interpret 6.0?
2. What do `us`, `sy`, `wa` and `st` in `top` indicate?
3. Why is `available`, not `free`, the important memory number?
4. How do you find what's filling a disk? What if `du` totals don't match `df`?
5. What do high `w_await` and `aqu-sz` in `iostat` mean?
6. What causes "too many open files," and how do you investigate it?
7. Why should you capture evidence before restarting a misbehaving service?

**Exercises**

1. Run the first-60-seconds checklist on a VM while it's idle, then while running a CPU
   stress test (`stress-ng --cpu 2`), then an I/O stress test. Compare outputs.
2. Fill a test disk to 100% and observe how Beacon, PostgreSQL and journald behave.
3. Create a deleted-but-open file (open a file in one shell with `tail -f`, delete it, write
   to it) and find it with `lsof +L1`.
4. Write a one-page runbook for "Beacon is slow" based on this chapter and Book III,
   Chapter 10.

**Interview-style questions**

- "A Linux server is slow. What do you check first?"
- "How do you find which process is using the most CPU, memory or disk I/O?"
- "The disk is full but you can't find the files. What's happening?"

---

## 14. Going deeper

- Brendan Gregg, [Linux Performance Analysis in 60,000 Milliseconds](https://netflixtechblog.com/linux-performance-analysis-in-60-000-milliseconds-accc10403c55)
  and his book *Systems Performance*.
- [brendangregg.com/linuxperf.html](https://www.brendangregg.com/linuxperf.html) — the
  famous diagram of Linux observability tools.
- Julia Evans, *Bite Size Linux* and *Debugging Tools* zines.

---

## Book VIII wrap-up

Beacon can now run on any Linux server: as a hardened systemd service behind Nginx with
automatic TLS, or as minimal, non-root, scanned container images orchestrated with Compose.
And when a server misbehaves, you have a method and a toolkit to find out why.

Running servers yourself is a valuable skill and a real option, but much of the operational
work (patching, scaling, failover, certificates, backups) can be handed to a cloud platform.
That's next.

**Next:** [Book IX — Azure, Cloud & DevOps](../09-cloud-devops/README.md).
