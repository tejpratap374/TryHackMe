	## Task 3: OS Enumeration

**Goal:** Identify the host's identity, kernel build, distribution release, scheduled tasks, and installed security/software packages to determine applicable exploits or privilege escalation paths.

- **Hostname (`hostname` or shell prompt):** Reveals the machine's name and often hints at its role or environment (e.g., `linux-enumeration`).
    
- **Kernel Version (`uname -r` or `uname -a` / `cat /proc/version`):** Shows the exact kernel build and architecture (e.g., `6.8.0-1017-aws`, indicating an AWS-flavored build). Kernel versions help check for known local privilege escalation flaws (such as Dirty COW or PwnKit).
    
- **OS / Distribution Version (`cat /etc/issue` or `/etc/os-release`):** Confirms the exact Linux distribution and release version (e.g., `Ubuntu 24.04.1 LTS`), which determines package versions and available patches.
    
- **Scheduled Tasks / Cron Jobs (`cat /etc/crontab`):** Lists system-wide scheduled jobs and intervals. Any script executed as `root` on a schedule (such as `/root/backup.sh` running every 5 minutes) is worth checking for weak permissions or writable directories/scripts.
    
- **Installed Packages (`dpkg -l | grep <package>`):** Queries installed Debian/Ubuntu packages and their full version strings (e.g., `apparmor` full version string from `dpkg -l`, which includes distribution-specific revision tags beyond what `apparmor_parser --version` shows).
    

### OS Enumeration Quick Reference

| **Check**      | **Command**                | **What to Look For**                            |
| -------------- | -------------------------- | ----------------------------------------------- |
| Hostname       | `hostname`                 | Host role or naming conventions                 |
| Kernel         | `uname -r` / `uname -a`    | Outdated kernel or cloud/custom flavor          |
| Distro Release | `cat /etc/issue`           | OS version (e.g., Ubuntu LTS release)           |
| Cron Jobs      | `cat /etc/crontab`         | Root-executed scripts or overly permissive jobs |
| Packages       | `dpkg -l \| grep apparmor` | Exact installed package & version string        |

## Task 4: User Enumeration

**Goal:** Understand your current user context, privileges, stored command history, environment configuration, and other accounts on the system.

- **Environment Variables (`env`):** Displays active shell and session variables (such as `PATH`, `LANG`, `USER`, `HOME`, and occasionally leaked secrets, tokens, or credentials). Poorly ordered or writable `PATH` entries can enable PATH hijacking.
    
- **Command History (`history` / `cat ~/.bash_history`):** Shows previously typed commands. Administrators or users sometimes accidentally enter passwords, API keys, or sensitive arguments directly on the command line.
    
- **Sudo Privileges (`sudo -l`):** A high-priority privilege escalation check showing which commands your current user can run as `root` (or another user) and whether `NOPASSWD` applies. Allowed binaries (like `/usr/bin/nmap` or other utilities with shell/file/script capabilities) can often be leveraged to escalate privileges.
    
- **System & Local Users (`cat /etc/passwd`):** Lists user accounts, UIDs, home directories, and login shells. Helps separate real/interactive user accounts from standard built-in service accounts (such as `list` for mailing list management).
    

### User Enumeration Quick Reference

|**Check**|**Command**|**What to Look For**|
|---|---|---|
|Environment|`env` / `env \| grep LANG`|`PATH` order, locale (`LANG`), leaked credentials|
|History|`history` / `cat ~/.bash_history`|Plaintext passwords, secrets, or past commands|
|Sudo Rights|`sudo -l`|Commands runnable as `root` (e.g., `/usr/bin/nmap`)|
|User Accounts|`cat /etc/passwd`|Human accounts vs. service accounts (e.g., `list`)|


## Task 5: Network Enumeration

**Goal:** Map the target host's network interfaces, internal subnets, open/listening ports, active connections, and running network services to identify pivoting opportunities or locally bound services that could be exploited for privilege escalation.

### 1. Network Interfaces (`ifconfig` / `ip addr`)

- **`ifconfig`** (part of `net-tools`) lists active network interfaces, IP addresses, netmasks, and MAC addresses.
    
- **What to look for:** Multiple interfaces or virtual bridges (such as `docker0` alongside `ens5` or `eth0`). Extra interfaces indicate the machine may be connected to additional internal subnets or container networks, making it a potential **pivoting point**.
    
- **Modern equivalent:** `ip addr` (or `ip a`).
    

### 2. Connections, Ports & Services (`netstat` / `ss`)

- **`netstat`** (part of `net-tools`) inspects active network sockets, listening ports, routing/interface stats, and associated processes.
    
- **Key `netstat` Flags:**
    
    - `-a`: Show all sockets (both listening servers and established connections).
        
    - `-l`: Show only **listening** ports (open and waiting for incoming connections).
        
    - `-t` / `-u`: Filter by **TCP** or **UDP** protocols respectively.
        
    - `-n`: Display **numeric** IP addresses and port numbers instead of resolving hostnames/service names (faster and avoids DNS delays).
        
    - `-p`: Show the **PID and program name** owning the socket (requires `root` privileges to view processes owned by other users).
        
    - `-s`: Display network traffic statistics grouped by protocol (`Ip`, `Icmp`, `Tcp`, `Udp`).
        
    - `-i`: Display the kernel interface table (MTU, packet counts, errors, drops).
        
    - `-o`: Display socket timers.
        
- **Common Combinations:**
    
    - `netstat -ano`: Lists all sockets numerically with timers—commonly used in write-ups to quickly spot ports listening only on `127.0.0.1` (localhost) vs. `0.0.0.0` (all interfaces).
        
    - `netstat -tpln`: Lists listening TCP ports numerically along with the backing service/PID.
        
- **Modern equivalent:** `ss` (e.g., `ss -tpln` replaces `netstat -tpln`).
    

### Network Enumeration Quick Reference

| **Task**              | **Legacy Command (net-tools)** | **Modern Equivalent** | **What to Look For**                                        |
| --------------------- | ------------------------------ | --------------------- | ----------------------------------------------------------- |
| List interfaces & IPs | `ifconfig`                     | `ip addr`             | Extra NICs, `docker0`, internal subnets for pivoting        |
| All sockets & ports   | `netstat -ano`                 | `ss -ano`             | Services bound to `127.0.0.1` (internal-only) vs. `0.0.0.0` |
| Listening TCP + PIDs  | `netstat -tpln`                | `ss -tpln`            | Exact service names/PIDs behind open ports                  |
| Protocol statistics   | `netstat -s`                   | `ss -s`               | Packet drops, forwarding status (`Forwarding: 1`)           |
| Interface table       | `netstat -i`                   | `ip -s link`          | Interface MTU, RX/TX errors or dropped packets              |


## Task 6: File Enumeration

**Goal:** Locate hidden files, sensitive credentials/flags, misconfigured permissions, writable directories, installed development tools, and SUID binaries that can aid privilege escalation.

### 1. Listing Hidden & Sensitive Files (`ls -la`)

- Always use **`ls -la`** during enumeration rather than `ls` or `ls -l`.
    
- `-l` shows long format (permissions, owner, group, size, timestamps), and **`-a` reveals hidden dotfiles** (names starting with `.`, such as `.secret.txt` or `.bash_history`), which are skipped by default.
    

### 2. Searching with `find`

- **`find`** recursively searches directories for files matching specific names, types, permissions, owners, times, or sizes.
    
- **Suppressing Permission Noise (`2>/dev/null`):** Searching from `/` triggers many `Permission denied` errors. Append `2>/dev/null` to redirect error messages to `/dev/null` and keep output readable.
    

#### Common `find` Patterns

- **By Name & Wildcards:**
    
    - `find . -name flag1.txt` — search current directory and subdirectories for `flag1.txt`.
        
    - `find /home -name flag1.txt` — search under `/home`.
        
    - `find / -name pass*.txt` — match filenames starting with `pass` and ending in `.txt` (e.g., `pass.txt`, `password.txt`).
        
- **By Type & Owner:**
    
    - `find / -type d -name config` — find directories (`-type d`) named `config`.
        
    - `find /home -user frank` — find all files owned by user `frank` under `/home`.
        
- **By Time & Size:**
    
    - `find / -mtime -10` / `find / -atime -10` — modified or accessed in the last **10 days**.
        
    - `find / -cmin -60` / `find / -amin -60` — status changed or accessed in the last **60 minutes**.
        
    - `find / -size +100M` — find files **larger than 100 MB** (`+` for larger than, `-` for smaller than).
        
- **By Permissions & SUID:**
    
    - `find / -type f -perm 0777` — regular files (`-type f`) with exact `0777` permissions.
        
    - `find / -perm -a=x` — files executable by all (`a=x`).
        
    - `find / -writable -type d 2>/dev/null` (or `-perm -222` / `-perm -o w`) — find world-writable directories.
        
    - `find / -perm -o x -type d 2>/dev/null` — find world-executable directories.
        
    - `find / -perm -u=s -type f 2>/dev/null` — find files with the **SUID** bit set, which run with the privileges of the file's owner (often `root`) rather than the invoking user.
        
- **Compilers & Scripting Languages:**
    
    - `find / -name perl*`, `find / -name python*`, `find / -name gcc*` — check whether interpreters or compilers are available on the host (useful for writing exploits or spawning shells).
        

#### Understanding `-perm` Modes

- `-perm mode` (no prefix): Requires an **exact** permission match (e.g., `-perm g=w` only matches when group write is the _sole_ permission bit set).
    
- `-perm -mode` (leading `-`): Requires **all** specified permission bits to be set.
    
- `-perm /mode` (leading `/`): Matches if **any** of the specified permission bits are set.
    

### File Enumeration Quick Reference

|**Check**|**Command**|**What to Look For**|
|---|---|---|
|Hidden files|`ls -la`|Dotfiles (`.secret.txt`, configs, history files)|
|Wildcard file search|`find / -name "*TryHackMe*" 2>/dev/null` or `find / -name pass*.txt`|Flags, credentials, backups, or named files|
|Large files|`find / -size +100M 2>/dev/null`|Archives, dumps, or oversized files|
|Writable folders|`find / -writable -type d 2>/dev/null`|World-writable directories (`-perm -222` / `-perm -o w`)|
|SUID binaries|`find / -perm -u=s -type f 2>/dev/null`|Binaries executing with owner (e.g., `root`) privileges|
|Dev/scripting tools|`find / -name python* 2>/dev/null`|`python`, `perl`, `gcc` available for exploitation|