# KLCP — Chapter 4 Labs

- [Lab 1 — CPU Architecture Verification](#lab-1--cpu-architecture-verification)
- [Lab 2 — Getting Started with Kali Linux](#lab-2--getting-started-with-kali-linux)
- [Final Command List](#final-command-list)
- [Status](#status)

---

# LAB 1 — CPU Architecture Verification

## Objective

Verify the CPU architecture and confirm whether the system supports 64-bit operation.

## Step 1 — Check Architecture

```bash
uname -m
```

Expected:

```text
x86_64
```

## Step 2 — Check CPU Flags

```bash
grep -m1 "flags" /proc/cpuinfo
```

Look for:

```text
lm
```

`lm` means Long Mode and indicates 64-bit capability.

## Step 3 — Automatically Check for 64-bit Support

```bash
grep -qP '^flags\s*:.*\blm\b' /proc/cpuinfo && echo 64-bit || echo 32-bit
```

Expected:

```text
64-bit
```

## Lab 1 Result

```text
Architecture: x86_64
CPU capability: 64-bit
Long Mode (lm): Present
```

---

# LAB 2 — Getting Started with Kali Linux

## Objective

Verify the Kali Linux download, check its authenticity and integrity, write the ISO to a USB device, and explore Kali Live boot options.

## Step 1 — Create Lab Directory

```bash
mkdir -p ~/lab2-kali
cd ~/lab2-kali
pwd
```

## Step 2 — Check Required Tools and Disk Space

```bash
gpg --version
sha256sum --version
df -h /
```

## Step 3 — Download SHA256 Checksums

```bash
wget -q https://cdimage.kali.org/current/SHA256SUMS
wget -q https://cdimage.kali.org/current/SHA256SUMS.gpg
ls -lh
```

## Step 4 — Import Kali Signing Key

```bash
wget -q -O - https://archive.kali.org/archive-key.asc | gpg --import
```

## Step 5 — Check Kali Key Fingerprint

```bash
gpg --fingerprint 827C8569F2518CC677FECA1AED65462EC8D5E4C5
```

Expected fingerprint:

```text
827C 8569 F251 8CC6 77FE CA1A ED65 462E C8D5 E4C5
```

## Step 6 — Verify SHA256SUMS Signature

```bash
gpg --verify SHA256SUMS.gpg SHA256SUMS
```

Expected:

```text
Good signature
```

> A "key is not certified with a trusted signature" warning is normal for a freshly imported key. Trust comes from the fingerprint check in Step 5.

## Step 7 — Find the ISO Hash

```bash
grep 'kali-linux-2026.2-live-amd64.iso$' SHA256SUMS
```

Expected hash:

```text
49e90e694d1b3dedd47f94afbe99dfdd5afb41c8462b638bbd332929769c773a
```

## Step 8 — Verify the ISO

```bash
sha256sum kali-linux-2026.2-live-amd64.iso
```

Then:

```bash
grep 'kali-linux-2026.2-live-amd64.iso$' SHA256SUMS | sha256sum -c
```

Expected:

```text
kali-linux-2026.2-live-amd64.iso: OK
```

## Step 9 — Identify the USB Device

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

Identify the USB carefully. Example:

```text
sdb    29G  disk  Kingston
└─sdb1 29G  part
```

## Step 10 — Unmount the USB

Replace `/dev/sdb1` if your USB uses a different device name.

```bash
sudo umount /dev/sdb1
```

Check again:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

## Step 11 — Write the Kali ISO to USB

> **WARNING:** This is destructive. Make absolutely sure `/dev/sdb` is the USB before running this command.

```bash
sudo dd if=~/lab2-kali/kali-linux-2026.2-live-amd64.iso of=/dev/sdb bs=4M status=progress conv=fsync
```

## Step 12 — Flush the Data

```bash
sync
```

Then check the USB:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL
```

## Step 13 — Boot Menu

Boot the computer/VM from the Kali USB.

Useful commands after Kali starts:

```bash
whoami
pwd
ls
```

Review the available Kali boot options:

```text
Live
Failsafe
Forensics mode
Persistence
Installer
Hardware Detection
Memory Diagnostic
```

## Step 14 — Live Storage Test

Create a small test image:

```bash
dd if=/dev/zero of=test.img bs=1M count=100
```

Check it:

```bash
ls -lh test.img
```

Remove it:

```bash
rm test.img
```

Create a file to test Live-session persistence:

```bash
echo "LAB 2 TEST" > ~/live-test.txt
```

Read it:

```bash
cat ~/live-test.txt
```

After reboot, check:

```bash
cat ~/live-test.txt
```

The file should not persist in a normal non-persistent Live session.

## Step 15 — Edit Kali Boot Options

At the Kali boot menu press:

```text
Tab
```

or, depending on the bootloader:

```text
e
```

Find and remove:

```text
quiet
```

Then boot. This allows more verbose boot messages to be displayed.

## Step 16 — Forensics Mode

Boot Kali using:

```text
Forensics mode
```

Check mounted filesystems:

```bash
mount
```

Check block devices:

```bash
lsblk
```

Inspect the Kali no-automount configuration:

```bash
cat /etc/X11/Xsession.d/52kali_noautomount
```

Important parameters:

```text
noswap
noautomount
```

---

# Final Command List

## Lab 1

```bash
uname -m
grep -m1 "flags" /proc/cpuinfo
grep -qP '^flags\s*:.*\blm\b' /proc/cpuinfo && echo 64-bit || echo 32-bit
```

## Lab 2

```bash
mkdir -p ~/lab2-kali
cd ~/lab2-kali
pwd

gpg --version
sha256sum --version
df -h /

wget -q https://cdimage.kali.org/current/SHA256SUMS
wget -q https://cdimage.kali.org/current/SHA256SUMS.gpg
ls -lh

wget -q -O - https://archive.kali.org/archive-key.asc | gpg --import

gpg --fingerprint 827C8569F2518CC677FECA1AED65462EC8D5E4C5

gpg --verify SHA256SUMS.gpg SHA256SUMS

grep 'kali-linux-2026.2-live-amd64.iso$' SHA256SUMS

sha256sum kali-linux-2026.2-live-amd64.iso

grep 'kali-linux-2026.2-live-amd64.iso$' SHA256SUMS | sha256sum -c

lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

sudo umount /dev/sdb1

sudo dd if=~/lab2-kali/kali-linux-2026.2-live-amd64.iso of=/dev/sdb bs=4M status=progress conv=fsync

sync

lsblk -o NAME,SIZE,TYPE,FSTYPE,MOUNTPOINTS,MODEL

whoami
pwd
ls

dd if=/dev/zero of=test.img bs=1M count=100
ls -lh test.img
rm test.img

echo "LAB 2 TEST" > ~/live-test.txt
cat ~/live-test.txt

mount
lsblk

cat /etc/X11/Xsession.d/52kali_noautomount
```

---

# Status

| Lab | Title | Status |
| --- | ----- | ------ |
| Lab 1 | CPU Architecture Verification | Completed |
| Lab 2 | Getting Started with Kali Linux | Completed |
