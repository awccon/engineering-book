# Linux Fundamentals

> **🔄 Current (as of October 2026):** Examples use Ubuntu Server 24.04 LTS or 26.04 LTS.
> Commands work on most distributions (Debian, Fedora/RHEL family with minor package
> manager differences) and inside most container images.

Most servers on the internet run Linux. So do almost all containers, including the
official .NET images, every Kubernetes node, and most cloud functions under the hood. Even
when you deploy to a managed platform like Azure App Service, your app is often running on
Linux, and when something goes wrong, you'll be reading Linux logs, checking Linux
permissions and running Linux commands in a container shell.

For a developer who has worked mostly on Windows with Visual Studio, Linux can feel like a
foreign country. This chapter is the phrasebook: the filesystem, users and permissions,
processes, the shell, and enough Bash to be productive and safe.

---

## 1. The problem: production isn't Windows (usually)

- .NET runs on Linux as a first-class platform, and Linux hosting is typically cheaper.
- Containers are Linux processes (Chapter 4).
- When your app misbehaves in production, the diagnostic tools are Linux tools: `ps`,
  `top`, `journalctl`, `ss`, `df`, `curl`.
- Configuration, permissions, file paths, line endings and case sensitivity all differ from
  Windows, and each difference has caused production outages.

---

## 2. The mental model: everything is a file, everything is a process

Linux is built on a few powerful ideas:

1. **Everything is a file.** Regular files, directories, devices (`/dev/sda`), process
   information (`/proc/1234/`), sockets, pipes. The same tools (`cat`, `ls`, permissions)
   work on all of them.
2. **Small tools that do one thing, composed with pipes.** `grep` filters, `sort` sorts,
   `wc` counts; `|` connects them.
3. **Everything runs as a user**, and permissions decide what each user can touch.
4. **Processes form a tree**, started by `init` (systemd), each with a parent, an owner, an
   environment and open files.

```text
 Hardware ◄── Kernel (processes, memory, filesystems, networking, permissions)
                ▲ system calls
 ┌──────────────┴───────────────────────────────────────────────┐
 │ User space: systemd, sshd, nginx, dotnet Beacon.Api.dll, bash  │
 └───────────────────────────────────────────────────────────────┘
```

The **kernel** manages hardware and enforces isolation. Everything else (shells, services,
your app) runs in **user space** and asks the kernel for resources via system calls.
Containers (Chapter 4) are user-space processes with extra kernel isolation.

---

## 3. The filesystem

There are no drive letters. Everything hangs off a single root, `/`:

| Path | Contains |
|---|---|
| `/` | Root of everything |
| `/home/<user>` | Users' home directories (`~`) |
| `/root` | The root user's home |
| `/etc` | System-wide configuration (`/etc/nginx/`, `/etc/systemd/`, `/etc/hosts`) |
| `/var` | Variable data: `/var/log` (logs), `/var/lib` (application state, databases) |
| `/opt` | Optional, self-contained software (a common place for your apps: `/opt/beacon`) |
| `/usr` | Installed programs and libraries (`/usr/bin`, `/usr/lib`) |
| `/tmp` | Temporary files (often cleared on reboot) |
| `/proc`, `/sys` | Virtual filesystems exposing kernel and process information |
| `/dev` | Devices |
| `/mnt`, `/media` | Mount points for other filesystems |

Differences from Windows that bite:

- **Paths are case-sensitive**: `Appsettings.json` and `appsettings.json` are different files.
  Code that works on Windows can fail on Linux because of a capitalization mismatch.
- **The separator is `/`**. In .NET, use `Path.Combine` and `Path.DirectorySeparatorChar`,
  never hard-coded `\`.
- **Line endings**: Linux uses `\n` (LF), Windows `\r\n` (CRLF). A shell script with CRLF
  endings fails with confusing errors (`/bin/bash^M: bad interpreter`). Set
  `.gitattributes` (`*.sh text eol=lf`).
- **Hidden files** start with a dot: `.bashrc`, `.ssh/`.

### Navigating and inspecting

```bash
pwd                         # where am I?
ls -la                      # list all files (including hidden), with details
cd /var/log                 # change directory; cd ~ or cd alone goes home; cd - goes back
cat file.txt                # print a file
less /var/log/syslog        # page through a file (q to quit, / to search)
head -n 20 file; tail -n 50 file
tail -f /var/log/nginx/access.log   # follow a file as it grows
file Beacon.Api.dll         # what kind of file is this?
du -sh /var/log/*           # sizes of things
df -h                       # free disk space per filesystem
find /opt/beacon -name '*.json' -mtime -1   # files modified in the last day
```

---

## 4. Users, groups and permissions

### Users and groups

Every process runs as a **user**; every file has an **owner** and a **group**. `root` (user
ID 0) can do anything. Regular users can only touch what permissions allow.

```bash
whoami                      # current user
id                          # user ID, group memberships
sudo <command>              # run a command as root (if you're allowed)
sudo -u beacon <command>    # run as another user
```

> **⚠️ What can go wrong:** Running your application as `root` means any vulnerability in it
> (or its dependencies) gives an attacker full control of the machine. Every service should
> run as its own **unprivileged user** with access only to what it needs: the same
> least-privilege principle as database roles (Book IV, Chapter 8).

### Permission bits

```bash
$ ls -l /opt/beacon
-rwxr-x---  1 beacon beacon  142336 Oct  7 09:12 Beacon.Api
-rw-r-----  1 beacon beacon    1830 Oct  7 09:12 appsettings.json
drwxr-x---  2 beacon beacon    4096 Oct  7 09:12 wwwroot
```

```text
 - rwx r-x ---
 │  │   │   └── others: no access
 │  │   └────── group (beacon): read, execute
 │  └────────── owner (beacon): read, write, execute
 └───────────── type: - file, d directory, l symlink
```

- **r** (4) read, **w** (2) write, **x** (1) execute (for directories, *x* means "can enter").
- Numeric form: `chmod 750 file` = `rwxr-x---`; `chmod 640 appsettings.json` = `rw-r-----`.

```bash
chmod 640 appsettings.Production.json        # owner read/write, group read, others nothing
chmod +x deploy.sh                           # make a script executable
chown beacon:beacon /opt/beacon -R           # change owner and group, recursively
```

A configuration file containing secrets readable by every user (`644`) is a common finding
in security audits.

---

## 5. Processes

A **process** is a running program with a **PID**, a parent, an owner, environment
variables, a working directory and open file descriptors.

```bash
ps aux                          # all processes
ps aux | grep Beacon            # find a process
pgrep -a dotnet                 # find by name, show command line
top                             # live view (or htop: friendlier, often needs installing)
kill 1234                       # ask process 1234 to stop (SIGTERM)
kill -9 1234                    # force kill (SIGKILL): last resort
```

### Signals

Processes communicate with the kernel and each other via **signals**:

| Signal | Meaning | Typical behavior |
|---|---|---|
| `SIGTERM` (15) | Please terminate | Graceful shutdown: finish requests, flush, exit |
| `SIGINT` (2) | Interrupt (Ctrl+C) | Same as above for interactive programs |
| `SIGKILL` (9) | Terminate now | Can't be caught: no cleanup at all |
| `SIGHUP` (1) | Hang up | Often "reload configuration" (Nginx) |

.NET's Generic Host handles `SIGTERM` and `SIGINT` by triggering graceful shutdown: hosted
services get their `stoppingToken` cancelled (Book III, Chapter 8), and Kestrel stops
accepting new requests and drains in-flight ones. That's why `SIGTERM` (not `SIGKILL`)
matters, and why containers and systemd send it first.

### Exit codes

Every process exits with a code: `0` for success, anything else for failure. Scripts,
systemd and container orchestrators use exit codes to decide whether something worked
(Book I, Chapter 8's CLI returned meaningful codes for this reason). `echo $?` shows the
last command's exit code. A container killed for using too much memory exits with `137`
(128 + 9: SIGKILL).

### Environment variables

```bash
env                                  # all environment variables
echo $HOME
export ASPNETCORE_ENVIRONMENT=Production
ASPNETCORE_URLS=http://127.0.0.1:5000 dotnet Beacon.Api.dll    # set just for this command
```

This is how configuration reaches .NET in production (Book III, Chapter 3):
`ConnectionStrings__Beacon` in the environment overrides `appsettings.json`.

---

## 6. The shell and Bash essentials

The **shell** (usually Bash, sometimes Zsh) interprets commands. A few concepts give you
most of its power.

### Pipes and redirection

```bash
command > out.txt           # stdout to a file (overwrite)
command >> out.txt          # append
command 2> err.txt          # stderr to a file
command > all.txt 2>&1      # both stdout and stderr
command < input.txt         # stdin from a file
cmd1 | cmd2                 # stdout of cmd1 becomes stdin of cmd2
```

A real example: the ten client IPs making the most requests to Nginx:

```bash
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head -10
```

Five tiny tools, composed. That's the Unix philosophy.

### The everyday text tools

| Tool | Does | Example |
|---|---|---|
| `grep` | Find lines matching a pattern | `grep -i 'error' app.log`, `grep -rn 'ConnectionStrings' /etc/beacon` |
| `sed` | Stream edit (replace text) | `sed 's/old/new/g' file` |
| `awk` | Column-based processing | `awk '{print $9}' access.log` (status codes) |
| `sort`, `uniq -c` | Sort, count duplicates | |
| `cut`, `tr`, `wc -l` | Columns, translate characters, count lines | |
| `jq` | Query JSON | `jq 'select(.Level == "Error")' log.json` |
| `curl` | HTTP requests | `curl -sS -o /dev/null -w '%{http_code}' http://localhost:5000/health/ready` |

Structured JSON logs (Book III, Chapter 3) plus `jq` make production logs queryable even
without a log platform:

```bash
journalctl -u beacon-api -o cat --since '10 min ago' \
  | jq -r 'select(.LogLevel == "Error") | "\(.Timestamp) \(.Message)"'
```

### Writing safe scripts

```bash
#!/usr/bin/env bash
set -euo pipefail          # exit on error, on unset variables, and on failures inside pipes
IFS=$'\n\t'

readonly APP_DIR=/opt/beacon
readonly RELEASE="${1:?usage: deploy.sh <release-dir>}"   # required argument with a message

echo "Deploying ${RELEASE} to ${APP_DIR}"
rsync -a --delete "${RELEASE}/" "${APP_DIR}/"
sudo systemctl restart beacon-api
curl --fail --retry 10 --retry-delay 2 --retry-all-errors http://127.0.0.1:5000/health/ready
echo "Deployed."
```

- `set -euo pipefail` is the single most important line: without it, Bash happily continues
  after a failed command.
- **Quote variables** (`"${VAR}"`): unquoted variables split on spaces and expand globs.
  `rm -rf $DIR/` with an empty `$DIR` is `rm -rf /`.
- Use **ShellCheck** (a linter for shell scripts) in your editor and CI.

> **🧭 When not to use Bash:** Shell scripts are great for gluing commands together (deploy
> steps, CI tasks, container entrypoints). Once a script needs data structures, error
> handling beyond "exit on failure," or more than ~100 lines, write it in a real language:
> a C# script (`dotnet run app.cs` in .NET 10), Python (Book X), or PowerShell (which runs on
> Linux too).

---

## 7. Packages and installing software

```bash
sudo apt update                     # refresh package lists (Debian/Ubuntu)
sudo apt install -y nginx jq        # install
sudo apt upgrade                    # upgrade installed packages
apt list --installed | grep dotnet
```

(Fedora/RHEL: `dnf`; Alpine: `apk`.) Install the .NET runtime from Microsoft's or Ubuntu's
package feeds: `sudo apt install -y aspnetcore-runtime-10.0`. For production, keep the
system patched: enable **unattended security upgrades** on servers, and rebuild container
images regularly (Chapter 6).

---

## 8. In practice: running Beacon on a Linux server

Let's run Beacon.Api on a fresh Linux VM (any cloud provider's smallest Ubuntu instance, or a
local VM). Chapter 2 makes it a proper service; for now, the manual steps teach the parts.

**1. Create a dedicated user** with no login shell:

```bash
sudo useradd --system --home /opt/beacon --shell /usr/sbin/nologin beacon
sudo mkdir -p /opt/beacon /etc/beacon
sudo chown beacon:beacon /opt/beacon
```

**2. Install the runtime:**

```bash
sudo apt update && sudo apt install -y aspnetcore-runtime-10.0
dotnet --list-runtimes          # Book I, Chapter 1: check what's installed
```

**3. Publish on your machine (or in CI) and copy:**

```bash
dotnet publish src/Beacon.Api -c Release -o out/api        # framework-dependent
rsync -avz out/api/ deploy@server:/tmp/beacon-release/
ssh deploy@server 'sudo rsync -a --delete /tmp/beacon-release/ /opt/beacon/ && sudo chown -R beacon:beacon /opt/beacon'
```

**4. Configuration with secrets**, readable only by root and the beacon group:

```bash
sudo tee /etc/beacon/beacon-api.env > /dev/null <<'EOF'
ASPNETCORE_ENVIRONMENT=Production
ASPNETCORE_URLS=http://127.0.0.1:5000
ConnectionStrings__Beacon=Host=db.internal;Database=beacon;Username=beacon_app;Password=change-me
EOF
sudo chown root:beacon /etc/beacon/beacon-api.env
sudo chmod 640 /etc/beacon/beacon-api.env
```

**5. Run it as the beacon user**, to check it starts:

```bash
sudo -u beacon bash -c 'set -a; source /etc/beacon/beacon-api.env; cd /opt/beacon && dotnet Beacon.Api.dll'
# in another terminal:
curl -sS http://127.0.0.1:5000/health/live
```

Notice the decisions: a dedicated unprivileged user, binding to `127.0.0.1` (not reachable
from outside; Nginx will be the public entry point in Chapter 3), secrets in a root-owned
file readable only by the app's group, and an explicit health check after starting.

Running it in a terminal isn't production. It stops when you log out, doesn't restart on
crash, and its logs go nowhere. Chapter 2 fixes all three with systemd.

---

## 9. What can go wrong

- **Running services as root.**
- **World-readable secrets** (`chmod 644` on files with passwords).
- **Case-sensitivity and path-separator bugs** from Windows-only development.
- **CRLF line endings** breaking shell scripts.
- **Unsafe scripts** without `set -euo pipefail` and quoting.
- **`kill -9` as a first resort**, skipping graceful shutdown.
- **Disk full** from logs or temp files: `df -h` and `du` should be reflexes.
- **Unpatched servers.**

---

## 10. How an experienced engineer thinks about this

- **Least privilege**: dedicated users, minimal permissions, nothing as root.
- **Compose small tools** to answer questions quickly: `grep`, `awk`, `sort`, `jq`, `curl`.
- **Prefer graceful signals** and make apps handle them.
- **Write scripts defensively**, and switch to a real language when they grow.
- **Develop with Linux in mind**: test on Linux in CI, mind case and paths.

---

## 11. Check yourself

**Questions**

1. What lives in `/etc`, `/var/log`, `/opt` and `/proc`?
2. Decode `-rw-r-----`. What does `chmod 750` set?
3. Why shouldn't your app run as root?
4. What's the difference between `SIGTERM` and `SIGKILL`? How does .NET respond to `SIGTERM`?
5. What does exit code 137 indicate?
6. What does `set -euo pipefail` do, and why quote variables?
7. How do environment variables reach .NET configuration?

**Exercises**

1. On a Linux VM or WSL, create the `beacon` user and directory layout from section 8 and run
   Beacon.Api as that user.
2. Write a one-liner that counts HTTP status codes in an Nginx access log.
3. Write a deploy script with `set -euo pipefail` and run ShellCheck on it.
4. Send `SIGTERM` to a running Beacon.Api and watch the graceful shutdown logs; then try
   `SIGKILL` and compare.

**Interview-style questions**

- "How do Linux file permissions work?"
- "How would you find which process is using the most memory on a server?"
- "What happens when you press Ctrl+C on a running process?"

---

## 12. Going deeper

- William Shotts, [*The Linux Command Line*](https://linuxcommand.org/tlcl.php) (free).
- [ShellCheck](https://www.shellcheck.net/)
- [Microsoft docs: Host ASP.NET Core on Linux](https://learn.microsoft.com/aspnet/core/host-and-deploy/linux-nginx)

**Next:** [Chapter 2 — Linux Services and Networking](02-linux-services-and-networking.md)
turns Beacon into a managed service and connects it to the network.
