# Kali Linux — Section 3 System Enumeration Labs (Commands)

Commands used in Labs 1–6 (Kali GNU/Linux Rolling 2026.2, VirtualBox VM).

## Lab 1 — Identify Your Kali System

```bash
uname -a
cat /etc/os-release
uname -r
hostname
hostnamectl
```

## Lab 2 — Desktop Environment

```bash
echo $XDG_CURRENT_DESKTOP
echo $DESKTOP_SESSION
loginctl show-session "$XDG_SESSION_ID" -p Type
```

## Lab 3 — APT Repository Enumeration

```bash
cat /etc/apt/sources.list
ls -la /etc/apt/
ls -la /etc/apt/sources.list.d/
grep -R "^deb" /etc/apt/ 2>/dev/null
cat /etc/apt/sources.list.d/kali.sources
```

## Lab 4 — Running Services & Network Exposure

```bash
systemctl --type=service --state=running
systemctl status ssh
systemctl is-enabled ssh
sudo ss -tulpn
```

## Lab 5 — Hardware Enumeration

```bash
lspci
lsusb
lscpu
free -h
df -h
```

## Lab 6 — Firmware

```bash
ls /lib/firmware
find /lib/firmware/ -type f | head -20
dmesg | grep -i firmware
sudo dmesg | head -30
```
