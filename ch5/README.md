# Cybersecurity Lab Report

Hands-on Linux security labs performed on a Kali Linux virtual machine.
A PDF version of this report is included: [Cybersecurity_Lab_Report.pdf](Cybersecurity_Lab_Report.pdf)

## Contents

| Lab | Chapter | Topic | Objective |
|---|---|---|---|
| [Lab 1](#lab-1--luks-encrypted-lvm-volume) | Lab 5 *(add chapter name)* | Encrypted storage with LUKS2, LVM and ext4 | Build an encrypted logical volume on a disk image, mount it and verify it works |
| [Lab 2](#lab-2--preseed-configuration-and-debconf-tools) | Lab 5 *(add chapter name)* | Preseed file and debconf tools | Create a minimal preseed file and install the debconf utilities |

---

## Lab 1 — LUKS-Encrypted LVM Volume

**Objective:** Build the stack `image file → loop device → LUKS → LVM → ext4 → mount point` and verify it.
**Environment:** Kali Linux VM, terminal, 500 MB image file.

### Step 1 — Prepare the workspace
![Workspace](images/lab1-01-workspace.png)

Created `~/lab5kali` and confirmed `cryptsetup` is installed.

### Step 2 — Attach the image to a loop device
![Loop reset](images/lab1-02-loop-reset.png)
![Loop result](images/lab1-03-loop-result.png)

`losetup` turns the image file into a block device (`/dev/loop0`) so it can be encrypted like a real disk.

### Step 3 — Format the device with LUKS
![luksFormat](images/lab1-04-luksformat.png)

`luksFormat` writes the LUKS header and sets the passphrase. It erases existing data, so it asks for confirmation.

### Step 4 — Verify the LUKS container
![isLuks](images/lab1-05-isluks.png)

`isLuks` answers through its exit code; `0` means success.

### Step 5 — Inspect the LUKS header
![luksDump](images/lab1-06-luksdump.png)

The header confirms LUKS2 with `aes-xts-plain64`, a 512-bit key and `argon2id` for key derivation.

### Step 6 — Unlock and create the LVM layers
![LVM](images/lab1-07-lvm.png)

The container is unlocked as `/dev/mapper/luks_lab`, then turned into a physical volume, volume group `lab_vg` and logical volume `lab_lv` (about 480 MB).

### Step 7 — Create the filesystem and mount
![ext4 and mount](images/lab1-08-ext4-mount.png)

The logical volume is formatted as ext4 and mounted at `/mnt/luks-lab`. `lsblk` shows the full stack.

### Step 8 — Test the volume
![Test](images/lab1-09-test.png)

The mount point is owned by root, so `sudo` is used. `| sudo tee` writes the file as root. The file reads back correctly.

### Commands used

| Command | Purpose |
|---|---|
| `cd ~/lab5kali`, `mkdir -p ~/lab5kali` | Create and enter the lab directory |
| `which cryptsetup` | Confirm the tool is installed |
| `sudo losetup -d /dev/loop0` | Detach a loop device (repeated for loop1 to loop3) |
| `sudo losetup --find --show lucks-lab-f.img` | Attach the image to a free loop device |
| `sudo losetup -a` | List active loop devices |
| `lsblk` | Show the block-device tree |
| `sudo cryptsetup luksFormat /dev/loop0` | Create the LUKS container |
| `sudo cryptsetup isLuks /dev/loop0` then `echo $?` | Check for a LUKS header (`0` = valid) |
| `sudo cryptsetup luksDump /dev/loop0` | Show LUKS header details |
| `sudo cryptsetup open /dev/loop0 luks_lab` | Unlock and map the container |
| `sudo pvcreate /dev/mapper/luks_lab` | Create the physical volume |
| `sudo vgcreate lab_vg /dev/mapper/luks_lab` | Create the volume group |
| `sudo lvcreate -l 100%FREE -n lab_lv lab_vg` | Create a logical volume using all free space |
| `sudo pvs` / `sudo vgs` / `sudo lvs` | List PV, VG and LV |
| `sudo mkfs.ext4 /dev/lab_vg/lab_lv` | Create the ext4 filesystem |
| `sudo blkid /dev/lab_vg/lab_lv` | Show filesystem type and UUID |
| `sudo mkdir -p /mnt/luks-lab` | Create the mount point |
| `sudo mount /dev/lab_vg/lab_lv /mnt/luks-lab` | Mount the volume |
| `df -h /mnt/luks-lab` | Check size and usage |
| `sudo touch /mnt/luks-lab/test.txt` | Create a test file |
| `echo "luks +vmlab" \| sudo tee /mnt/luks-lab/test.txt` | Write to the file as root |
| `cat /mnt/luks-lab/test.txt` | Read the file back |

### Key concepts

| Term | Meaning |
|---|---|
| Loop device | A file used as a block device |
| LUKS | Linux disk-encryption standard |
| LVM | Pools storage (PV, VG) and carves flexible logical volumes (LV) |
| ext4 | Journaling filesystem on the logical volume |

### Results and takeaways
- The image was encrypted with LUKS2, layered with LVM, formatted as ext4 and mounted.
- A test file was written and read from the encrypted volume.
- A device needs a LUKS header before it can be opened; encryption sits below LVM and the filesystem.

---

## Lab 2 — Preseed Configuration and debconf Tools

**Objective:** Create a basic preseed file and use the debconf tools to look at installer settings.
**Environment:** Kali Linux VM, nano.

### Step 1 — Create the preseed file
![Preseed](images/lab2-01-preseed.png)

A preseed file answers installer questions in advance. This one uses `kali-rolling`, disables root login and creates the user `labuser`.

### Step 2 — Install the debconf utilities
![debconf-utils](images/lab2-02-debconf-utils.png)

`debconf-utils` provides `debconf-get-selections`, which exports debconf answers in preseed format.

### Step 3 — Query the installer selections
![Permission](images/lab2-03-permission.png)

The debconf databases are root-protected, so a normal user cannot read them. Reading this data requires elevated privileges.

### Commands used

| Command | Purpose |
|---|---|
| `mkdir preslab`, `cd preslab` | Create and enter the lab directory |
| `nano preseed.cfg` | Create and edit the preseed file |
| `cat preseed.cfg` | Display the file |
| `sudo apt install debconf-utils` | Install the debconf tools |
| `which debconf-get-selections` | Confirm the command is available |
| `debconf-get-selections --installer` | Query installer selections (needs elevated privileges) |

### Results and takeaways
- A four-line preseed file was created.
- `debconf-utils` was installed.
- Preseed lines follow the form `d-i question type value`.
- debconf data is protected by file permissions.

- ## Lab 3 — Kernel Command Line and Boot Messages

**Objective:** Read the parameters the kernel was booted with and look at the early boot messages.
**Environment:** Kali Linux VM (VirtualBox), terminal.

### Step 1 — Read the kernel command line
![Command line](images/lab3-01-cmdline.png)

`/proc/cmdline` holds the boot parameters. `tr ' ' '\n'` puts each parameter on its own line, and `grep -E` highlights the ones of interest. `dmesg | head -30` then shows the first kernel messages.

| Parameter | Meaning |
|---|---|
| `BOOT_IMAGE=/boot/vmlinuz-6.19.14+kali-amd64` | Kernel image that was loaded |
| `root=UUID=...` | Device used as the root filesystem |
| `ro` | Root filesystem is mounted read-only at first |
| `quiet` | Reduces boot messages on screen |
| `splash` | Shows the boot splash screen |

### Step 2 — Filter the boot messages
![dmesg filter](images/lab3-02-dmesg-filter.png)

`dmesg | grep -iE 'boot|kernel|command'` keeps only the lines about boot, kernel and command. The kernel command line appears again at the start of the log, and systemd lines show kernel modules and file systems being set up.

### Commands used

| Command | Purpose |
|---|---|
| `cat /proc/cmdline` | Show the kernel boot parameters |
| `cat /proc/cmdline \| tr ' ' '\n'` | Show one parameter per line |
| `cat /proc/cmdline \| grep -E 'quiet\|splash\|root\|ro\|rw'` | Highlight selected parameters |
| `dmesg \| head -30` | Show the first kernel messages |
| `dmesg \| grep -iE 'boot\|kernel\|command'` | Filter boot messages by keyword |

### Results and takeaways
- The kernel was booted from `vmlinuz-6.19.14+kali-amd64` with `ro quiet splash`.
- `dmesg` shows hardware detection and boot steps, including the VirtualBox environment.
- `/proc/cmdline` and `dmesg` are the first places to look when checking how a system booted.

---

---

## Before publishing

The LUKS header dump (Lab 1, Step 5) shows a UUID and salts. They belong to a disposable lab image, but redact them if you reuse the setup.
