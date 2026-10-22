# Chapter 8 Mini Lab — Linux Security Investigation (Kali Linux)

A beginner-level, hands-on lab in which I investigated my own Kali Linux virtual machine the way a junior SOC analyst would: identifying the system, reviewing processes, checking listening ports, listing running services, reading error-level journal entries, and inspecting the firewall. Every command, output, error, and conclusion below comes from my actual terminal screenshots. Nothing was installed, enabled, flushed, or reconfigured during the lab.

> This repository documents hands-on learning and practical evidence. It is not an official certification project or an employer-approved assessment.

**Author:** [Your Name] · **Lab date:** 22 October 2026 (from screenshot filenames and journal timestamps)

## Table of Contents

1. [Lab Overview](#lab-overview)
2. [Environment and Requirements](#environment-and-requirements)
3. [Complete Command Documentation](#complete-command-documentation)
   - [Task 1 — Identify the system](#task-1--identify-the-system)
   - [Task 2 — Running processes](#task-2--running-processes)
   - [Task 3 — Listening ports](#task-3--listening-ports)
   - [Task 4 — Running services](#task-4--running-services)
   - [Task 5 — System errors](#task-5--system-errors)
   - [Task 6 — Firewall inspection](#task-6--firewall-inspection)
4. [Final Challenge Answers](#final-challenge-answers)
5. [Consolidated Findings](#consolidated-findings)
6. [Troubleshooting Log](#troubleshooting-log)
7. [Evidence and Screenshots](#evidence-and-screenshots)
8. [Security and Safety Notes](#security-and-safety-notes)
9. [Lessons Learned](#lessons-learned)

---

## Lab Overview

**What was investigated:** The identity, running software, network exposure, logged errors, and firewall posture of a single Kali Linux VM.

**Why it matters:** These are the first questions in almost any host triage: *Who am I? What is running? What is listening? What is failing? What is the firewall doing?* Each answer comes from a read-only command.

**Activities performed:** 18 command executions (13 distinct successful commands plus 5 failed attempts caused by typos or missing privileges), all documented below.

**Learning objectives:**

- Identify the user, host, and kernel involved in an investigation.
- Read process snapshots sorted by CPU and memory without assuming heavy use means malicious.
- Check for listening TCP/UDP sockets with `ss`.
- List running systemd services and inspect one service's status.
- Read error-priority journal entries and judge their significance.
- Read a firewall ruleset (nftables and iptables views) without changing it.

## Environment and Requirements

| Item | Value | Basis |
|---|---|---|
| OS | Kali Linux | Prompt `kali@kali`, `+kali` kernel suffix |
| Kernel | `6.19.14+kali-amd64` | `uname -r` |
| Architecture | amd64 (64-bit x86) | Kernel string |
| User / host | `kali` / `kali` | `whoami`, `hostname` |
| Shell | zsh | `/usr/bin/zsh` in `ps` output |
| Terminal | QTerminal | `/usr/bin/qterminal` in `ps` output |
| Desktop | Xfce | `xfwm4`, `xfce4-panel`, `xfdesktop` in `ps` output |
| Virtualization | VirtualBox guest (inferred) | `VBoxClient`, `VBoxDRMClient`, `virtualbox-guest-utils.service` |
| Privileges | Normal user with `sudo` | `[sudo] password for kali:` |
| Firewall tooling | `nft` and `iptables` both at `/usr/sbin/` | `command -v` |

**Tools used (all pre-installed):** `ps`, `head`, `ss`, `systemctl`, `journalctl`, `nft`, `iptables`, `sudo`. No packages were installed.

## Complete Command Documentation

Legend: **[RO]** = read-only inspection. No command in this lab modified system state.

### Task 1 — Identify the system

#### `whoami`

**Purpose:** Prints the effective username of the current shell. **[RO]**

**Syntax breakdown:** No arguments.

```bash
whoami
```

**Actual output:**

```text
kali
```

**Output explanation:** The shell runs as the account `kali`.

**What the output tells us:** The investigation began as the unprivileged `kali` user, not root.

**Why it matters:** Knowing which account you act as determines what you can read and whether `sudo` is needed.

**Limitations:** It does not show group memberships or sudo rights (sudo was later shown to work).

#### `hostname`

**Purpose:** Prints the system's hostname. **[RO]**

```bash
hostname
```

**Actual output:**

```text
kali
```

**What the output tells us:** The machine's hostname is `kali`, matching the journal entries (`... kali kernel: ...`).

**Why it matters:** In an investigation, tying evidence to the correct host is essential.

**Limitations:** Says nothing about IP address or domain.

#### `uname -r`

**Purpose:** Prints the running kernel release. **[RO]**

**Syntax breakdown:** `uname` prints system information; `-r` selects only the kernel release string.

```bash
uname -r
```

**Actual output:**

```text
6.19.14+kali-amd64
```

**Output explanation:** `6.19.14` is the kernel version; `+kali` indicates Kali's build; `amd64` indicates the 64-bit x86 architecture.

**What the output tells us:** The running kernel is a Kali-packaged 64-bit kernel.

**Why it matters:** Kernel version is the starting point for checking known vulnerabilities and patch status.

**Limitations:** It shows the *running* kernel only; it does not show whether newer kernels are installed or whether this one has pending fixes.

### Task 2 — Running processes

#### `ps aux --sort=-%cpu | head -n 10`

**Purpose:** Lists the highest-CPU processes in a snapshot. **[RO]**

**Syntax breakdown:**

- `ps aux` — all processes from all users, in user-oriented format (`a` = all users' processes, `u` = user format, `x` = include processes without a terminal).
- `--sort=-%cpu` — sort by CPU percentage; the leading `-` means descending.
- `|` — sends `ps` output to the next command.
- `head -n 10` — keeps the first 10 lines (the header plus 9 processes).

```bash
ps aux --sort=-%cpu | head -n 10
```

**Actual output** (long command lines abbreviated with `...`; full text in the screenshot):

```text
USER   PID  %CPU %MEM   VSZ    RSS   TTY   STAT START TIME COMMAND
root   741  0.8  7.5  425768 152560 tty7  Ssl+ 12:37 0:14 /usr/lib/xorg/Xorg :0 -seat seat0 -auth /var/run/lightdm/root/:0 -nolisten tcp vt7 -novtswitch
kali  1107  0.2  0.1  215384   3344 ?     Sl   12:37 0:04 /usr/bin/VBoxClient --draganddrop
root    46  0.2  0.0       0      0 ?     I    12:36 0:03 [kdamond.0]
kali  9402  0.2  3.4  589016  70080 ?     Sl   12:51 0:01 /usr/bin/qterminal
kali  1178  0.1  6.9  888436 139988 ?     Sl   12:37 0:02 xfwm4
kali  9409  0.1  0.3   14200   8020 pts/0 Ss   12:51 0:01 /usr/bin/zsh
kali  1281  0.1  2.0  293932  41912 ?     Sl   12:37 0:02 ...wrapper-2.0 ...libcpugraph.so ... cpugraph CPU Graph
kali  1283  0.1  1.5  277204  30500 ?     Sl   12:37 0:02 ...wrapper-2.0 ...libgenmon.so ... genmon Generic Monitor
root   589  0.0  0.1  352944   2924 ?     Sl   12:37 0:01 /usr/bin/VBoxDRMClient
```

**Output explanation:**

- The top entry is **Xorg** (PID 741, root), the X display server, at 0.8 % CPU and 7.5 % memory. Its arguments include `-nolisten tcp`, which tells Xorg not to accept X11 connections over TCP.
- **VBoxClient --draganddrop** and **VBoxDRMClient** are VirtualBox guest-integration helpers.
- **[kdamond.0]** is a bracketed name, which indicates a kernel thread (state `I` = idle).
- **qterminal** and **zsh** (pts/0) are the terminal and shell I was typing in; they started at 12:51, later than the 12:37 desktop session.
- **xfwm4**, **cpugraph**, and **genmon** are Xfce window-manager and panel components.
- `STAT` codes: `S` sleeping, `l` multi-threaded, `s` session leader, `+` foreground process group.

**What the output tells us:** CPU use is very low everywhere (highest 0.8 %), and the busiest processes are the display stack, VirtualBox integration, and the terminal session itself. Nothing in this snapshot is unexplained.

**Why it matters:** A process snapshot is the baseline for spotting anomalies. High CPU alone is not proof of malice; context (name, path, user, parent) decides.

**Limitations:** A single snapshot. `%CPU` in `ps` is an average over the process lifetime, not an instantaneous reading, and short-lived processes would be missed.

#### `ps aux --sort=-%mem | head -n 10`

**Purpose:** Same as above, sorted by memory percentage. **[RO]**

**Syntax breakdown:** Identical to the CPU version except `--sort=-%mem` (descending by memory).

**Execution 1 — failed:**

```bash
ps aux --sort=-%mem | head -n 10
```

```text
error: unknown sort specifier

Usage:
 ps [options]
...
For more details see ps(1).
```

**Explanation:** `ps` rejected the sort key. The command text looks identical to the working one in the screenshot at this resolution, so the exact typo (for example a mistyped character) is **not confirmed**. The retry fixed it.

**Execution 2 — succeeded** (the screenshots cut the output across two images; one row is not visible, see Limitations):

```text
USER   PID  %CPU %MEM   VSZ    RSS   TTY  STAT START TIME COMMAND
root   741  0.8  7.5  425768 152560 tty7 Ssl+ 12:37 0:14 /usr/lib/xorg/Xorg :0 ...
kali  1178  0.1  6.9  888436 139988 ?    Sl   12:37 0:03 xfwm4
kali  1546  0.0  3.1  604972  63144 ?    Sl   12:37 0:00 /usr/bin/python3 /usr/bin/blueman-applet
kali  1237  0.0  2.9  698236  58884 ?    Sl   12:37 0:00 xfdesktop
kali  1534  0.0  2.3   76536  47472 ?    S    12:37 0:00 /usr/bin/python3 /usr/share/system-config-printer/applet.py
kali  1220  0.0  2.1  646016  44000 ?    Sl   12:37 0:00 xfce4-panel
kali  1580  0.0  2.1  670096  43620 ?    Sl   12:37 0:00 nm-applet
kali  1255  0.0  2.1  653144  42988 ?    Sl   12:37 0:00 ...wrapper-2.0 ...libwhiskermenu.so ... whiskermenu
```

**Output explanation:** The largest memory user is Xorg (7.5 %, about 152 MB RSS), then xfwm4 (6.9 %). The remaining entries are desktop applets: Bluetooth (`blueman-applet`), printer configuration, network (`nm-applet`), and the Whisker application menu.

**What the output tells us:** Memory use is dominated by the graphical desktop, with no process above 7.5 %. This is expected for a desktop Kali VM.

**Why it matters:** Memory-sorted views reveal processes such as miners or leaking services that CPU sorting can miss.

**Limitations:** Only eight process rows are visible but `head -n 10` should print nine. Based on the 3.4 % memory value for `qterminal` in the CPU listing, the missing row is *probably* `qterminal`, but that is an inference, not something visible in the evidence. `%MEM` is also a snapshot.

**Mission answer (one process explained):** `xfwm4` is the Xfce window manager. It draws window borders and handles moving and resizing windows. Its 6.9 % memory use is normal for a graphical session.

### Task 3 — Listening ports

#### `sudo ss -tulpn`

**Purpose:** Lists listening network sockets together with the owning processes. **[RO]**

**Syntax breakdown:**

- `sudo` — runs as root so process information for all sockets can be shown.
- `ss` — socket statistics (modern replacement for `netstat`).
- `-t` TCP, `-u` UDP, `-l` listening sockets only, `-p` show owning process, `-n` numeric ports and addresses (no name lookups).

```bash
sudo ss -tulpn
```

**Actual output:**

```text
[sudo] password for kali:
Netid  State  Recv-Q  Send-Q  Local Address:Port  Peer Address:Port  Process
```

**Output explanation:** Only the column header was printed. No socket rows followed. (The password prompt appears; the password itself is never displayed.)

**What the output tells us:** At that moment, with root visibility, there were **no listening TCP or UDP sockets**. This is consistent with the service list (no SSH, web, or file-sharing server running) and with Xorg's `-nolisten tcp` flag.

**Why it matters:** Listening ports define the host's network-facing attack surface. A clean result is a good baseline.

**Limitations:** It does not cover non-listening (outbound or established) connections, nor sockets that open later. It also says nothing about what is reachable from outside; that depends on firewall and network layout.

**Mission answers:**

1. Any TCP/UDP sockets listening? **No.**
2. Port numbers? **None appeared.**
3. Associated processes? **None, because there were no listeners.**
4. Does a listening port automatically mean internet exposure? **No** — it depends on the bind address, the firewall, and NAT/network design.

### Task 4 — Running services

#### `systemctl --type=service --state=running --pager`

**Purpose / syntax:** Intended to list running services; the flag `--pager` is not valid. **[RO]**

```bash
systemctl --type=service --state=running --pager
```

```text
systemctl: unrecognized option '--pager'
```

**Explanation:** `systemctl` has `--no-pager` (disable the pager), not `--pager`. This was a typo; nothing ran. Fixed by the next command.

#### `systemctl --type=service --state=running --no-pager`

**Purpose:** Lists currently running systemd services. **[RO]**

**Syntax breakdown:** `--type=service` limits output to service units; `--state=running` shows only units whose low-level state is running; `--no-pager` prints directly instead of opening `less`.

```bash
systemctl --type=service --state=running --no-pager
```

**Actual output:**

```text
UNIT                              LOAD   ACTIVE SUB     DESCRIPTION
accounts-daemon.service           loaded active running Accounts Service
colord.service                    loaded active running Manage, Install and Generate Color Profiles
cron.service                      loaded active running Regular background program processing daemon
dbus.service                      loaded active running D-Bus System Message Bus
getty@tty1.service                loaded active running Getty on tty1
haveged.service                   loaded active running Entropy Daemon based on the HAVEGE algorithm
lightdm.service                   loaded active running Light Display Manager
ModemManager.service              loaded active running Modem Manager
NetworkManager.service            loaded active running Network Manager
polkit.service                    loaded active running Authorization Manager
rtkit-daemon.service              loaded active running RealtimeKit Scheduling Policy Service
systemd-journald.service          loaded active running Journal Service
systemd-logind.service            loaded active running User Login Management
systemd-timesyncd.service         loaded active running Network Time Synchronization
systemd-udevd.service             loaded active running Rule-based Manager for Device Events and Files
systemd-userdbd.service           loaded active running User Database Manager
udisks2.service                   loaded active running Disk Manager
upower.service                    loaded active running Daemon for power management
user@1000.service                 loaded active running User Manager for UID 1000
virtualbox-guest-utils.service    loaded active running Virtualbox guest utils

20 loaded units listed.
```

**Output explanation:** `LOAD loaded` = unit definition read correctly; `ACTIVE active` / `SUB running` = the service is currently running. Twenty units are listed.

**What the output tells us:** The services are typical for a desktop VM: display (`lightdm`), networking (`NetworkManager`, `systemd-timesyncd`), logging (`systemd-journald`), scheduling (`cron`), authorization (`polkit`), and VirtualBox integration. No SSH, web server, database, or file-sharing service is in the list.

**Why it matters:** The service list is the other half of attack-surface review. It agrees with `ss`: no network server is listening.

**Limitations:** Only *running* services are shown, not stopped, failed, or disabled ones.

**Mission answer (one service explained):** `NetworkManager.service` manages network interfaces and connections (Wi-Fi, Ethernet, DHCP settings).

#### `systemctl status ssh --no-pager`

**Purpose:** Shows the state of the SSH server unit. **[RO]**

```bash
systemctl status ssh --no-pager
```

**Actual output:**

```text
o ssh.service - OpenBSD Secure Shell server
     Loaded: loaded (/usr/lib/systemd/system/ssh.service; disabled; preset: disabled)
     Active: inactive (dead)
       Docs: man:sshd(8)
             man:sshd_config(5)
```

**Output explanation:**

- The hollow circle `o` indicates an inactive unit.
- `Loaded: loaded` — the unit file exists at `/usr/lib/systemd/system/ssh.service`, so the OpenSSH server **is installed**.
- `disabled; preset: disabled` — it does not start at boot, and the distribution preset also leaves it disabled.
- `Active: inactive (dead)` — it is not running now.

**What the output tells us:** SSH server software is present but neither running nor enabled. This matches the empty `ss` output (nothing listening on port 22).

**Why it matters:** Distinguishing *installed* from *enabled* from *running* is a core administrative skill. Per the lab instructions, SSH was not enabled or started.

**Limitations:** It does not show whether the unit could start successfully, or its configuration.

### Task 5 — System errors

#### `journalctl -b -p err -n 20 --no-pager`

**Purpose:** Shows recent error-priority log entries from the current boot. **[RO]**

**Syntax breakdown:** `-b` current boot only; `-p err` priority `err` and more severe; `-n 20` last 20 entries (fewer exist); `--no-pager` print directly.

```bash
journalctl -b -p err -n 20 --no-pager
```

**Actual output** (7 entries):

```text
Oct 22 12:36:56 kali kernel: vmwgfx 0000:00:02.0: [drm] *ERROR* vmwgfx seems to be running on an unsupported hypervisor.
Oct 22 12:36:56 kali kernel: vmwgfx 0000:00:02.0: [drm] *ERROR* This configuration is likely broken.
Oct 22 12:36:56 kali kernel: vmwgfx 0000:00:02.0: [drm] *ERROR* Please switch to a supported graphics device to avoid problems.
Oct 22 12:37:19 kali lightdm[951]: gkr-pam: unable to locate daemon control file
Oct 22 12:37:38 kali obexd[1953]: Unable to acquire registry: Error calling StartServiceByName for org.gnome.evolution.dataserver.Sources5: Unit evolution-source-registry.service not found.
Oct 22 12:37:38 kali obexd[1953]: Unable to acquire registry: Error calling StartServiceByName for org.gnome.evolution.dataserver.Sources5: Unit evolution-source-registry.service not found.
```

(The screenshot shows seven lines because two of the long messages wrap; the `obexd` message appears twice.)

**Output explanation:**

- **vmwgfx (kernel):** The VMware virtual GPU driver reports it is running on an unsupported hypervisor. Combined with the VirtualBox guest tools, this likely means the VM's graphics controller is presented as a VMware-compatible adapter. This is a **hypothesis**, not confirmed.
- **gkr-pam (lightdm):** The GNOME Keyring PAM module could not find its daemon control file during login. It relates to keyring integration at login.
- **obexd:** The Bluetooth OBEX service tried to start an Evolution address-book registry service that is not installed (`Unit ... not found`).

**What the output tells us:** All errors are environment- or integration-related (virtual graphics, keyring, optional desktop component). None indicates authentication failure, service compromise, or unusual activity.

**Why it matters:** Reading the journal is how you separate noise from real problems. Errors need context; they are not proof of an attack.

**Limitations:** Only priority `err` or worse is shown; warnings are excluded. The ongoing impact on the system was not tested.

**Mission answer:** The `vmwgfx` error looks like an *environment/driver configuration* issue rather than an attack. Further investigation would mean reviewing the VM's display settings (recommended check — not executed).

#### `sudo journalctl -b -p err -n 20 --n-pager`

```bash
sudo journalctl -b -p err -n 20 --n-pager
```

```text
journalctl: unrecognized option '--n-pager'
```

**Explanation:** Typo for `--no-pager`. Nothing was run.

#### `sudo journalctl -b -p err -n 20 --no-pager`

**Purpose:** The lab's fallback with root privileges. **[RO]**

```bash
sudo journalctl -b -p err -n 20 --no-pager
```

**Actual output:** The same seven entries as the non-sudo run.

**What the output tells us:** The non-sudo command already had access to these error entries, so `sudo` was not required here and revealed nothing additional.

**Limitations:** Why the normal user has access is not shown (group membership was not checked).

### Task 6 — Firewall inspection

#### `command -v nft`

**Purpose:** Checks whether `nft` exists and prints its path. **[RO]**

**Syntax breakdown:** `command` is a shell builtin; `-v` prints the path (or name) the shell would run.

```bash
command -v nft
```

```text
/usr/sbin/nft
```

**What the output tells us:** The nftables command-line tool is installed at `/usr/sbin/nft`.

#### `sudo nft list rulest`

```bash
sudo nft list rulest
```

```text
Error: syntax error, unexpected string
list rulest
     ^^^^^^
```

**Explanation:** `rulest` is a typo for `ruleset`. `nft` pointed its caret at the unexpected word. Nothing was changed.

#### `sudo nft list ruleset`

**Purpose:** Prints the complete active nftables ruleset. **[RO]** (`list` only displays; it does not edit or flush.)

```bash
sudo nft list ruleset
```

**Actual output (key excerpts; the full output spans screenshots 4–7 and was not fully captured, see Limitations):**

```text
# Warning: table ip filter is managed by iptables-nft, do not touch!
table ip filter {
    chain ufw-before-input {
        iifname "lo" counter packets 4 bytes 200 accept
        ct state related,established counter packets 12 bytes 1570 accept
        ct state invalid counter packets 0 bytes 0 drop
        ... (ICMP, DHCP, mDNS 224.0.0.251:5353, SSDP 239.255.255.250:1900 rules)
        counter packets 1 bytes 635 jump ufw-not-local
        counter packets 1 bytes 635 jump ufw-user-input
    }
    chain ufw-after-input {
        udp dport 137 / 138, tcp dport 139 / 445 -> jump ufw-skip-to-policy-input
    }
    chain ufw-after-logging-input {
        limit rate 3/minute burst 10 packets counter packets 1 bytes 635 log prefix "[UFW BLOCK] "
    }
    chain INPUT   { type filter hook input   priority filter; policy drop;   ... }
    chain OUTPUT  { type filter hook output  priority filter; policy accept; ... }
    chain FORWARD { type filter hook forward priority filter; policy drop;   ... }
    chain ufw-user-input {
        ip saddr 10.0.2.0/24 tcp dport 445 counter packets 0 bytes 0 accept
    }
    chain ufw-track-output {
        ip protocol udp ct state new counter packets 12 bytes 1453 accept
    }
}
# Warning: table ip6 filter is managed by iptables-nft, do not touch!
table ip6 filter {
    chain ufw6-before-input { ... }   (output continues beyond the captured screenshots)
```

**Output explanation:**

- The warning means these tables were created through `iptables-nft`; they should be managed with iptables/UFW tools, not edited with raw `nft`.
- Chains named `ufw-*` show that the rules were created by **UFW (Uncomplicated Firewall)**.
- **INPUT policy drop:** incoming traffic not explicitly accepted is dropped. **FORWARD policy drop:** no routing of traffic through the host. **OUTPUT policy accept:** outgoing traffic is allowed.
- Accepted inbound traffic includes loopback and `related,established` replies (return traffic for connections the host started).
- `ufw-user-input` contains one user rule: accept TCP port 445 from `10.0.2.0/24`. Its counter is 0 packets.
- **Counters:** INPUT saw 17 packets (4 loopback, 12 established, 1 other). The one remaining 635-byte packet passed `ufw-not-local` (counted under the *multicast* return rule), reached `ufw-after-logging-input` (the `[UFW BLOCK]` log rule, counter 1 / 635 bytes), and was dropped by the default policy.

**What the output tells us:** Inbound traffic is default-deny with only specific allowances, which agrees with `ss` finding no listeners. One multicast-type packet was blocked and was matched by the logging rule.

**Why it matters:** Reading the firewall tells you what the host *would* allow in, independent of what is listening. Counters show actual traffic that hit each rule.

**Limitations:** `ufw status` was not run, so UFW's active state is inferred from the rules, not confirmed. Whether the `[UFW BLOCK]` entry was actually written to a log was not checked. The IPv6 table was cut off in the screenshots. The `10.0.2.0/24` subnet is commonly VirtualBox's default NAT range, but that is not confirmed by the evidence. Because nothing listens on port 445, this allow rule currently has no practical effect.

#### `command -v iptables`

```bash
command -v iptables
```

```text
/usr/sbin/iptables
```

**What the output tells us:** `iptables` is installed.

#### `command iptables -L -n -v` (without sudo)

```bash
command  iptables -L -n -v
```

```text
iptables v1.8.13 (nf_tables): Could not fetch rule set generation id: Permission denied (you must be root)
```

**Explanation:** The lab's `command -v iptables` was apparently altered to `command iptables ...`, which *runs* iptables rather than just locating it (a typing slip). As a normal user, iptables could not read the rule set. This confirms root privileges are needed, and also shows the backend is `nf_tables` (iptables 1.8.13).

#### `sudo iptables -L -n -v`

**Purpose:** Lists the IPv4 filter table rules numerically with counters. **[RO]**

**Syntax breakdown:** `-L` list rules; `-n` numeric (no DNS/port-name lookups); `-v` verbose (packet/byte counters, interfaces).

```bash
sudo  iptables -L -n -v
```

**Actual output (key excerpts, screenshots 8–11):**

```text
Chain INPUT (policy DROP 1 packets, 635 bytes)
 pkts bytes target                    prot opt in out source    destination
   17  2405 ufw-before-logging-input   all  -- *  *   0.0.0.0/0 0.0.0.0/0
   17  2405 ufw-before-input           all  -- *  *   0.0.0.0/0 0.0.0.0/0
    1   635 ufw-after-input            all  -- *  *   0.0.0.0/0 0.0.0.0/0
    ...
Chain FORWARD (policy DROP 0 packets, 0 bytes)
Chain OUTPUT (policy ACCEPT 3 packets, 120 bytes)
Chain ufw-after-input (1 references)
    0 0 ufw-skip-to-policy-input udp dpt:137 / udp dpt:138 / tcp dpt:139 / tcp dpt:445 / udp dpt:67 / udp dpt:68 / ADDRTYPE BROADCAST
Chain ufw-before-input (1 references)
    4  200 ACCEPT all -- lo
   12 1570 ACCEPT all  ctstate RELATED,ESTABLISHED
    0    0 DROP   all  ctstate INVALID
    0    0 ACCEPT icmp types 3, 11, 12, 8
    0    0 ACCEPT udp spt:67 dpt:68
    1  635 ufw-not-local all
    0    0 ACCEPT udp 224.0.0.251 dpt:5353
    0    0 ACCEPT udp 239.255.255.250 dpt:1900
    1  635 ufw-user-input all
Chain ufw-not-local (1 references)
    1  635 RETURN all  ADDRTYPE match dst-type MULTICAST
Chain ufw-track-output (1 references)
    0    0 ACCEPT tcp ctstate NEW
   12 1453 ACCEPT udp ctstate NEW
Chain ufw-user-input (1 references)
    0    0 ACCEPT tcp  10.0.2.0/24  0.0.0.0/0  tcp dpt:445
```

**Output explanation:** This is the same ruleset as the nftables view, shown in classic iptables format. The policy counters match: INPUT dropped exactly 1 packet (635 bytes). OUTPUT's default-accept handled 3 packets (120 bytes). Unreferenced chains such as `ufw-user-limit` show `(0 references)` and are unused. `ufw-not-local` shows the 635-byte packet was classified as **MULTICAST**, which supports the earlier interpretation.

**What the output tells us:** Two independent views (nft and iptables) agree: default-deny inbound, return traffic allowed, one narrow user rule for TCP 445 from `10.0.2.0/24`.

**Why it matters:** Cross-checking two tools builds confidence, and the iptables output is easier for many admins to read.

**Limitations:** Same as nft: IPv4 only here, UFW status not directly confirmed, no live-traffic test performed.

## Final Challenge Answers

| # | Question | Answer | Supported by |
|---|---|---|---|
| 1 | What does `ss -tulpn` help investigate? | Listening ports and network sockets | Task 3 |
| 2 | Does a listening port prove internet exposure? | No — other network and firewall controls matter | Firewall default-deny in Task 6 |
| 3 | What is `journalctl` for? | Inspecting system journal logs | Task 5 |
| 4 | First step with an unfamiliar process? | Investigate its name, executable, user, and behavior | Task 2 |

## Consolidated Findings

**Confirmed from evidence**

- System: Kali Linux, user/host `kali`, kernel `6.19.14+kali-amd64`.
- Processes: low CPU everywhere (max 0.8 %); top memory use is Xorg (7.5 %) and xfwm4 (6.9 %). All visible processes are recognizable desktop or VirtualBox components.
- Network: **no listening TCP or UDP sockets**.
- Services: 20 running services; no SSH/web/database/file-sharing service. `ssh.service` is installed but disabled and inactive.
- Errors: 7 error-level journal entries, all related to virtual graphics, keyring, or an optional Evolution component.
- Firewall: UFW-style rules via `iptables-nft`; INPUT/FORWARD default drop, OUTPUT default accept; one allow rule for TCP 445 from `10.0.2.0/24`; one 635-byte multicast packet dropped and counted by the `[UFW BLOCK]` log rule.

**Not established**

- Whether UFW is formally "active" (`ufw status` not run).
- Cause of the `vmwgfx` error (only hypothesized).
- The identity of the missing row in the memory-sorted `ps` output.
- The remainder of the IPv6 ruleset.

**Recommended additional checks — not executed:** `sudo ufw status verbose`; `sudo journalctl -b -p warning -n 50 --no-pager`; `ip -br addr`; review the VM's graphics controller setting.

## Troubleshooting Log

| Issue | Evidence | Diagnosis | Action Taken | Final Status |
|---|---|---|---|---|
| `ps` sort rejected | `error: unknown sort specifier` | Typo in the sort key (exact typo not visible) | Re-ran `ps aux --sort=-%mem \| head -n 10` | Resolved |
| Invalid systemctl flag | `unrecognized option '--pager'` | Should be `--no-pager` | Re-ran with `--no-pager` | Resolved |
| Invalid journalctl flag | `unrecognized option '--n-pager'` | Typo for `--no-pager` | Re-ran with `--no-pager` | Resolved |
| nft syntax error | `unexpected string ... list rulest` | `rulest` should be `ruleset` | Re-ran `sudo nft list ruleset` | Resolved |
| iptables permission denied | `Could not fetch rule set generation id: Permission denied (you must be root)` | Ran as normal user; also `command iptables` was typed instead of `command -v iptables` | Re-ran with `sudo` | Resolved |
| `vmwgfx` kernel errors | 3 `*ERROR*` lines at 12:36:56 | Likely VM graphics-controller mismatch (hypothesis) | None (observation only) | Not verified |
| `gkr-pam` and `obexd` errors | Journal lines at 12:37:19 and 12:37:38 | Missing keyring daemon file / uninstalled Evolution registry unit | None | Not verified |

## Evidence and Screenshots

Suggested repository layout (a proposal; copy the files into place so the links work):

```text
chapter8-linux-security-investigation/
├── README.md
├── report/
│   └── Chapter8_Lab_Report.pdf
└── screenshots/
    ├── Screenshot_2026-10-22_201525.png
    ├── Screenshot_2026-10-22_201545.png
    ├── Screenshot_2026-10-22_201607.png
    ├── Screenshot_2026-10-22_201625.png
    ├── Screenshot_2026-10-22_201643.png
    ├── Screenshot_2026-10-22_201659.png
    ├── Screenshot_2026-10-22_201713.png
    ├── Screenshot_2026-10-22_201743.png
    ├── Screenshot_2026-10-22_201801.png
    ├── Screenshot_2026-10-22_201815.png
    └── Screenshot_2026-10-22_201836.png
```

| Evidence | Content |
|---|---|
| `..._201525.png` | Task 1 commands; CPU-sorted `ps`; failed and start of successful memory-sorted `ps` |
| `..._201545.png` | Rest of memory-sorted `ps`; `ss -tulpn`; `--pager` error; running services |
| `..._201607.png` | SSH status; `journalctl` runs (including the `--n-pager` typo); `command -v nft`; `nft` typo |
| `..._201625.png` – `..._201713.png` | `sudo nft list ruleset` (IPv4 table; IPv6 table starts at `201713`) |
| `..._201743.png` | `command -v iptables`; permission-denied run; start of `sudo iptables -L -n -v` |
| `..._201801.png` – `..._201836.png` | Remaining iptables chains and counters |

Example (once files are in place):

```markdown
![Task 1 and 2](screenshots/Screenshot_2026-10-22_201525.png)
```

Screenshots are unaltered terminal captures; errors were kept in the documentation.

## Security and Safety Notes

- All commands used here are **read-only**. Review any command before running it with `sudo`.
- Do not run `nft flush ruleset`, `iptables -F`, or change firewall policies on a system you rely on; the lab explicitly said not to change anything.
- The nft warning (`managed by iptables-nft, do not touch!`) is a reminder to use the owning tool (UFW/iptables) for changes.
- Do not publish passwords or tokens. The sudo password was typed at a prompt and does not appear in the evidence.
- Avoid enabling services (such as SSH) only for a lab; every running service increases attack surface.

## Lessons Learned

- Empty output can be a result: `ss -tulpn` returning only headers means no listeners, which agrees with the service list and Xorg's `-nolisten tcp`.
- "Installed", "enabled", and "running" are three separate states (`ssh.service` is installed but neither enabled nor running).
- Errors in a journal need classification, not panic: the three error sources here were virtualization, keyring, and a missing optional component.
- Firewall policy and listening sockets are separate layers; reading both gives the full exposure picture.
- Packet counters (17 in, 1 dropped, 635 bytes) let you trace exactly what a firewall did.
- Many of my failures were typos in flags (`--pager`, `--n-pager`, `rulest`). Reading the exact error text points straight to the fix.
