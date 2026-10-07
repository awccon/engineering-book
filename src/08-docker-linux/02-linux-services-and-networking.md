# Linux Services and Networking

Chapter 1 ended with Beacon.Api running in a terminal: it stops when you log out, doesn't
restart when it crashes, and its logs disappear. Production services need a supervisor
that starts them at boot, restarts them on failure, stops them gracefully, captures their
logs and limits their privileges. On modern Linux, that's **systemd**.

Services also need to be reachable (and *unreachable*) over the network. This chapter
covers systemd and journald, SSH for remote access, and the networking fundamentals you
need to reason about ports, DNS, firewalls and connections.

---

## 1. The problem: keeping a process alive and reachable

A production service must:

- start automatically at boot, after its dependencies (network, database),
- restart when it crashes, but not in a tight loop,
- shut down gracefully on deploys and reboots,
- have its logs captured, rotated and searchable,
- run with minimal privileges,
- listen on the right address and port, and be reachable only by who should reach it.

---

## 2. systemd: the mental model

**systemd** is the first process the kernel starts (PID 1) on most distributions. It
manages **units**: services, timers, sockets, mounts. Each unit is a configuration file
describing what to run and how.

```text
 systemd (PID 1)
 ├── sshd.service
 ├── nginx.service
 ├── postgresql.service
 ├── beacon-api.service     ← our unit
 └── beacon-retention.timer → beacon-retention.service (scheduled job)
```

### A service unit for Beacon.Api

```ini
# /etc/systemd/system/beacon-api.service
[Unit]
Description=Beacon API
After=network-online.target
Wants=network-online.target

[Service]
Type=notify                                   # .NET signals "ready" via systemd integration
User=beacon
Group=beacon
WorkingDirectory=/opt/beacon
ExecStart=/usr/bin/dotnet /opt/beacon/Beacon.Api.dll
EnvironmentFile=/etc/beacon/beacon-api.env

Restart=on-failure                            # restart if it crashes...
RestartSec=5                                  # ...after 5 seconds...
StartLimitIntervalSec=300
StartLimitBurst=5                             # ...but give up after 5 failures in 5 minutes

KillSignal=SIGTERM
TimeoutStopSec=30                             # grace period for in-flight requests (matches HostOptions)

# Hardening: least privilege at the OS level
NoNewPrivileges=true
ProtectSystem=strict                          # filesystem read-only except listed paths
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/beacon
CapabilityBoundingSet=
LimitNOFILE=65535                             # enough file descriptors for many connections

[Install]
WantedBy=multi-user.target
```

`Type=notify` works because the app calls `UseSystemd()`:

```bash
dotnet add src/Beacon.Api package Microsoft.Extensions.Hosting.Systemd
```

```csharp
builder.Host.UseSystemd();   // no-op when not running under systemd
```

It tells systemd when the app is actually ready (after startup and configuration validation),
and formats console logs for the journal.

### Managing services

```bash
sudo systemctl daemon-reload                 # after creating or editing unit files
sudo systemctl enable --now beacon-api       # start now and at every boot
systemctl status beacon-api                  # state, PID, recent log lines
sudo systemctl restart beacon-api
sudo systemctl stop beacon-api
systemctl list-units --type=service --state=failed
systemd-analyze security beacon-api          # score the unit's hardening
```

### Timers: scheduled jobs

systemd **timers** replace cron for scheduled tasks, with logging and dependency handling.
Beacon's nightly retention job (Book IV, Chapter 8):

```ini
# /etc/systemd/system/beacon-retention.service
[Unit]
Description=Beacon data retention job
[Service]
Type=oneshot
User=beacon
EnvironmentFile=/etc/beacon/beacon-api.env
ExecStart=/usr/bin/dotnet /opt/beacon/Beacon.Cli.dll retention --batch-size 1000
```

```ini
# /etc/systemd/system/beacon-retention.timer
[Unit]
Description=Run Beacon retention nightly
[Timer]
OnCalendar=*-*-* 02:30:00
RandomizedDelaySec=15min
Persistent=true                     # run at next boot if the machine was off at 02:30
[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now beacon-retention.timer
systemctl list-timers
```

---

## 3. Logs with journald

systemd's **journal** captures stdout and stderr of every service, plus kernel and system
messages, in a structured, indexed store:

```bash
journalctl -u beacon-api                       # all logs for the service
journalctl -u beacon-api -f                    # follow live
journalctl -u beacon-api --since "1 hour ago" -p err   # errors in the last hour
journalctl -u beacon-api -b                    # since last boot
journalctl -u beacon-api -o json-pretty -n 5   # structured fields
journalctl --disk-usage
sudo journalctl --vacuum-time=14d              # trim old entries
```

The journal rotates and caps its size (configure in `/etc/systemd/journald.conf`), which
avoids the classic "log file filled the disk" outage. In production, logs are also shipped
to a central platform (OpenTelemetry, Book IX, Chapter 6), but `journalctl` on the box is
often the fastest first look during an incident.

Traditional text logs live in `/var/log` (Nginx's `access.log` and `error.log`, `auth.log`
for logins), rotated by **logrotate**.

---

## 4. SSH: secure remote access

**SSH** (Secure Shell) gives you an encrypted terminal on a remote machine, and is the
foundation for remote administration, file copying and Git over SSH.

### Keys, not passwords

```bash
ssh-keygen -t ed25519 -C "you@example.com"          # creates ~/.ssh/id_ed25519 (private) and .pub (public)
ssh-copy-id deploy@server.example.com               # appends your public key to ~/.ssh/authorized_keys on the server
ssh deploy@server.example.com
```

The private key never leaves your machine (protect it with a passphrase and an agent). The
server stores only public keys.

### Hardening the SSH server

```text
# /etc/ssh/sshd_config.d/hardening.conf
PasswordAuthentication no          # keys only: password brute-forcing is constant on the internet
PermitRootLogin no                 # log in as a normal user, then sudo
KbdInteractiveAuthentication no
AllowUsers deploy
```

```bash
sudo sshd -t && sudo systemctl reload ssh     # validate config before reloading: don't lock yourself out
```

Additional layers: restrict SSH by firewall to known IPs or a VPN, use a **bastion host**
(jump box), or avoid public SSH entirely with cloud tools (Azure Bastion, serial console)
or an identity-aware proxy. Watch `/var/log/auth.log` (or `journalctl -u ssh`) to see how
constantly the internet tries to log in.

### Useful SSH features

```bash
ssh -L 5433:db.internal:5432 deploy@bastion      # local port forward: reach a private DB at localhost:5433
scp file.txt deploy@server:/tmp/                 # copy a file
rsync -avz ./out/ deploy@server:/tmp/release/    # efficient sync (only changed files)
```

```text
# ~/.ssh/config: name your hosts
Host beacon-prod
  HostName 203.0.113.10
  User deploy
  IdentityFile ~/.ssh/id_ed25519
```

---

## 5. Networking fundamentals

### IP addresses, ports and sockets

- An **IP address** identifies a network interface (`203.0.113.10`, `10.0.1.5`, `::1`).
- A **port** (0–65535) identifies a service on that address. Ports below 1024 are
  privileged (binding needs root or a capability).
- A **socket** is one endpoint of a connection: (IP, port, protocol). A TCP connection is
  identified by the four-tuple (client IP, client port, server IP, server port).

### Listening addresses matter

| Bind address | Reachable from |
|---|---|
| `127.0.0.1` / `localhost` | **Only this machine** |
| A private IP (`10.0.1.5`) | The private network |
| `0.0.0.0` / `::` (all interfaces) | **Everywhere the network allows** |

Beacon.Api binds to `127.0.0.1:5000`, so only Nginx on the same machine can reach it. A
database or Redis bound to `0.0.0.0` on a public server is how data leaks happen (Book IV,
Chapter 8).

### Inspecting the network

```bash
ss -tlnp                                  # TCP listening sockets, numeric, with processes
ss -tnp state established '( dport = :5432 )'   # connections to PostgreSQL
ip addr                                   # interfaces and addresses
ip route                                  # routing table
curl -v http://127.0.0.1:5000/health/ready
nc -vz db.internal 5432                   # can I open a TCP connection?
```

### DNS

**DNS** translates names to addresses:

```bash
dig beacon.example.com          # query DNS (A/AAAA records, TTL)
dig +short beacon.example.com
getent hosts db.internal        # resolve using the system's resolver (includes /etc/hosts)
cat /etc/resolv.conf            # which DNS servers this machine uses
```

Record types you'll configure: **A** (IPv4), **AAAA** (IPv6), **CNAME** (alias to another
name), **TXT** (verification, SPF/DKIM for email), **MX** (mail). **TTL** controls caching:
lower it *before* a migration so changes propagate quickly.

> **⚠️ What can go wrong:** "It's always DNS." Stale cached records, a wrong record during a
> migration, internal names that don't resolve from a container, or an expired domain are
> behind a surprising share of outages. `dig` from the failing machine is a first step.

### Firewalls

Allow only what's needed. On Ubuntu, **ufw** is a friendly front end to the kernel's
packet filter:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow from 198.51.100.0/24 to any port 22 proto tcp   # SSH only from the office/VPN range
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status verbose
```

Cloud environments add **network security groups** / **security lists** in front of the VM
(Book IX, Chapter 5). Use both: defense in depth.

### TCP connection lifecycle (enough to debug)

```text
 Client                                   Server
   SYN  ─────────────────────────────────►
        ◄───────────────────────── SYN-ACK
   ACK  ─────────────────────────────────►      connection established (1 round trip)
   ... data ...
   FIN  ─────────────────────────────────►      close
        ◄──────────────────────────── ACK / FIN
   ACK  ─────────────────────────────────►
   (client side waits in TIME_WAIT for a while)
```

Many sockets in `TIME_WAIT` on a client usually means connections aren't being reused: the
`new HttpClient()`-per-request problem from Book III, Chapter 1.

---

## 6. Troubleshooting connectivity, step by step

> **🔍 Investigation: "Beacon.Api can't reach the database."** Work up the layers:
> 1. **DNS**: `getent hosts db.internal`. Does the name resolve, to the expected IP?
> 2. **Routing / reachability**: `nc -vz db.internal 5432`. Timeout suggests a firewall or
>    routing problem; "connection refused" means the host is reachable but nothing listens.
> 3. **Listening**: on the DB host, `ss -tlnp | grep 5432`. Is PostgreSQL bound to the right
>    interface (`listen_addresses`), not just `127.0.0.1`?
> 4. **Firewalls**: `ufw status` on both hosts; cloud network rules.
> 5. **TLS**: certificate errors in the app logs (`sslmode=verify-full` needs the right CA
>    and hostname).
> 6. **Authentication**: `pg_hba.conf` rules, credentials, roles.
> 7. **Capacity**: connection limits exhausted (`too many connections`).
>
> Each step's output tells you which layer to fix, rather than guessing.

---

## 7. In practice: Beacon as a systemd service

On the server from Chapter 1:

```bash
sudo mkdir -p /var/lib/beacon && sudo chown beacon:beacon /var/lib/beacon
sudo cp beacon-api.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable --now beacon-api
systemctl status beacon-api
journalctl -u beacon-api -f
```

Test the behaviors that matter:

```bash
# Crash recovery: kill the process; systemd restarts it after 5 s
sudo kill -9 "$(systemctl show -p MainPID --value beacon-api)"
systemctl status beacon-api           # Active: activating (auto-restart) → active (running)

# Graceful shutdown: in-flight requests complete
sudo systemctl restart beacon-api     # SIGTERM, up to 30 s, then start

# Hardening score
systemd-analyze security beacon-api   # aim for a low "exposure" value

# Network exposure: the API listens only on localhost
ss -tlnp | grep 5000                  # 127.0.0.1:5000, not 0.0.0.0:5000
```

Lock down the machine:

```bash
sudo ufw default deny incoming && sudo ufw allow from <your-ip> to any port 22 proto tcp
sudo ufw allow 80,443/tcp && sudo ufw enable
```

At this point Beacon.Api is a resilient service that isn't reachable from the internet.
Chapter 3 puts Nginx in front of it with TLS.

---

## 8. What can go wrong

- **No supervisor**: services running in `screen`/`tmux` or with `nohup`, not restarted
  after crashes or reboots.
- **Restart loops** hiding a crash (always check `systemctl status` and the journal).
- **Services bound to `0.0.0.0`** that should be local or private.
- **Password SSH** exposed to the internet; root login enabled.
- **Locking yourself out** by misconfiguring SSH or the firewall (test in a second session
  before closing the first).
- **Logs filling the disk.**
- **Short stop timeouts** cutting off in-flight requests and background work.
- **DNS caching and TTL surprises** during migrations.

---

## 9. How an experienced engineer thinks about this

- **Every long-running process has a supervisor** that restarts it and captures its logs.
- **Harden at every layer**: unprivileged user, systemd sandboxing, firewall, minimal
  listening addresses.
- **Keys and least exposure for SSH.**
- **Debug networks layer by layer**: name, route, port, firewall, TLS, auth, capacity.
- **Test failure behavior** (kill, restart, reboot) before you need it.

---

## 10. Check yourself

**Questions**

1. What does systemd do for a service that running it in a terminal doesn't?
2. What do `Restart=on-failure`, `TimeoutStopSec` and `Type=notify` do?
3. Why prefer SSH keys over passwords, and why disable root login?
4. What's the difference between binding to `127.0.0.1` and `0.0.0.0`?
5. What do A, CNAME and TXT DNS records do? What is TTL for?
6. How do you tell a firewall problem from "nothing is listening"?
7. What does a large number of `TIME_WAIT` sockets suggest?

**Exercises**

1. Install Beacon.Api as a systemd service with the hardening options above, and check its
   `systemd-analyze security` score.
2. Create the retention timer and verify it in `systemctl list-timers` and the journal.
3. Set up SSH key authentication on a VM, disable passwords, and confirm password login fails.
4. Use `ss`, `nc` and `dig` to diagnose a deliberately broken connection (wrong port,
   firewall rule, bad hostname).

**Interview-style questions**

- "How do you run a .NET application as a service on Linux?"
- "A service can't connect to the database. How do you troubleshoot?"
- "How do you secure SSH access to a server?"

---

## 11. Going deeper

- [systemd.service](https://www.freedesktop.org/software/systemd/man/latest/systemd.service.html)
  and [systemd.exec](https://www.freedesktop.org/software/systemd/man/latest/systemd.exec.html) manuals
- [Microsoft docs: Host ASP.NET Core on Linux with Nginx](https://learn.microsoft.com/aspnet/core/host-and-deploy/linux-nginx)
- Julia Evans' networking zines and blog (jvns.ca) — friendly explanations of DNS, TCP and more.

**Next:** [Chapter 3 — Nginx, Reverse Proxies and TLS](03-nginx-reverse-proxies-and-tls.md)
puts Beacon on the internet, securely.
