


# 📂 Linux Filesystem Hierarchy — Final Comprehensive Summary

---

## 📚 Overview

The Linux filesystem is structured as a **single inverted tree**, starting at the **root directory `/`**. Everything (files, devices, sockets, processes) is represented as a file under `/`.

---

## 🔑 Root Directory `/`

The top-level directory containing all system files, user files, devices, and mounts.

---

## 📁 Key Directories Under `/`

### 1. **/bin**

* Essential **user binaries** (commands) needed for booting and single-user mode.
* Examples: `ls`, `cp`, `mv`, `cat`, `grep`, `bash`.

### 2. **/sbin**

* Essential **system binaries** for administration, mainly used by root.
* Examples: `init`, `fsck`, `ifconfig`, `reboot`.

### 3. **/boot**

* Files needed for booting the system.
* Kernel (`vmlinuz`), initramfs (`initrd.img`), GRUB files.

### 4. **/dev**

* Device files (special files that interface with hardware/peripherals).
* Examples:

    * `/dev/sda` → first SATA disk
    * `/dev/null` → null device
    * `/dev/tty` → terminal devices

### 5. **/etc**

* System-wide **configuration files**.
* Examples:

    * `/etc/passwd` → user accounts
    * `/etc/fstab` → filesystem mount table
    * `/etc/hosts` → hostname mappings

### 6. **/home**

* User home directories.
* Example: `/home/alice`, `/home/bob`

### 7. **/lib**, **/lib64**

* Essential **shared libraries** for programs in `/bin` and `/sbin`.
* Examples:

    * `/lib/libc.so` → C library
    * `/lib/modules` → kernel modules

### 8. **/media**

* Mount point for **removable media**.
* Examples: `/media/usb`, `/media/cdrom`

### 9. **/mnt**

* Generic temporary mount point (often used for manual mounts).
* Example: `/mnt/data`

### 10. **/opt**

* Optional or **third-party software** packages.
* Example: `/opt/google/chrome/`

### 11. **/proc**

* **Virtual filesystem** providing process and kernel info.
* Examples:

    * `/proc/cpuinfo`
    * `/proc/meminfo`
    * `/proc/[PID]/`

### 12. **/root**

* Home directory of the **root user** (not `/` itself).

### 13. **/run**

* Volatile runtime data (replaces `/var/run` in modern systems).
* Example: process IDs, sockets.

### 14. **/srv**

* Site-specific **service data** (e.g., web/ftp server files).
* Example: `/srv/www/`

### 15. **/sys**

* Virtual filesystem exposing **kernel/device info**.
* Example: `/sys/class/net/`

### 16. **/tmp**

* Temporary files (writable by anyone, usually cleared on reboot).
* Example: session files, sockets.

### 17. **/usr**

* Secondary hierarchy for **user applications and read-only data**.
* Often very large.

    * `/usr/bin` → non-essential user binaries
    * `/usr/sbin` → non-essential system binaries
    * `/usr/lib` → libraries
    * `/usr/include` → header files
    * `/usr/share` → architecture-independent data (man pages, docs, icons)
    * `/usr/local` → locally installed software

### 18. **/var**

* Variable files (expected to grow).

    * `/var/log` → log files
    * `/var/spool` → queued tasks (print, mail)
    * `/var/tmp` → temp files that persist across reboots
    * `/var/cache` → cached data

---

## 🗂 Special Notes

* **Everything is a file**: devices, processes, sockets.
* **Hard vs Soft links**: Multiple references to files.
* **Mount points**: Additional filesystems mounted under `/`.
* **Virtual filesystems**: `/proc` and `/sys` don’t hold real files, but kernel-generated info.

---

✅ **Mnemonic Tip**:

* **bin/sbin** → binaries
* **etc** → configuration
* **var** → variable data
* **usr** → user apps
* **lib** → libraries
* **dev** → devices
* **proc/sys** → system + kernel info


---

## 🌳 Full Directory Tree with Annotations

```
/                               # root of the entire filesystem tree
├─ bin/                         # essential user commands (ls, cp, mv, cat, bash)
├─ sbin/                        # essential system binaries for root (init, reboot, fsck)
├─ boot/                        # boot loader + kernel
│   ├─ vmlinuz                  # Linux kernel
│   ├─ initrd.img               # initial RAM disk
│   └─ grub/                    # GRUB bootloader files
├─ dev/                         # device files (interface to hardware/peripherals)
│   ├─ sda                      # first hard disk
│   ├─ null                     # discard output
│   ├─ tty                      # terminals
│   └─ random                   # random number generator
├─ etc/                         # system-wide configuration files
│   ├─ passwd                   # user accounts
│   ├─ shadow                   # secure user passwords
│   ├─ fstab                    # filesystem mount table
│   └─ hosts                    # static hostname mappings
├─ home/                        # user home directories
│   ├─ alice/                   # home dir for user "alice"
│   └─ bob/                     # home dir for user "bob"
├─ lib/                         # essential shared libraries for /bin and /sbin
├─ lib64/                       # 64-bit libraries (on 64-bit systems)
├─ media/                       # mount points for removable media (USB, CD-ROM)
├─ mnt/                         # generic temporary mount point (manual use)
├─ opt/                         # optional / third-party software
├─ proc/                        # virtual filesystem with process & kernel info
│   ├─ cpuinfo                  # CPU details
│   ├─ meminfo                  # memory details
│   └─ [PID]/                   # info for each running process
├─ root/                        # root user’s home directory
├─ run/                         # volatile runtime data (PID files, sockets) — cleared on reboot
├─ srv/                         # service-specific data (web, ftp, etc.)
├─ sys/                         # virtual filesystem exposing kernel & device info
├─ tmp/                         # temporary files (world-writable, cleared on reboot)
├─ usr/                         # user applications and read-only data (very large)
│   ├─ bin/                     # non-essential user commands (awk, gcc, python)
│   ├─ sbin/                    # non-essential system binaries (httpd, named)
│   ├─ lib/                     # libraries for /usr/bin and /usr/sbin
│   ├─ include/                 # header files (C, C++)
│   ├─ share/                   # arch-independent data (docs, man pages, icons)
│   └─ local/                   # locally installed software (safe from package manager)
└─ var/                         # variable data files (changes frequently)
    ├─ log/                     # log files (syslog, dmesg, auth.log)
    ├─ spool/                   # queued tasks (mail, print jobs)
    ├─ tmp/                     # temporary files that persist across reboot
    └─ cache/                   # application cache data
```

---

## 🔑 Key Principles

* **Everything is a file**: devices, sockets, processes, directories.
* **Essential binaries** → `/bin`, `/sbin`.
* **Configuration** → `/etc`.
* **User data** → `/home`.
* **Variable data** → `/var`.
* **Kernel + process info** → `/proc`, `/sys`.
* **Boot files** → `/boot`.
* **Temporary storage** → `/tmp`, `/var/tmp`.
* **3rd party software** → `/opt`, `/usr/local`.

---

✅ This summary gives you:

* A **tree-structured overview** (easy to visualize).
* **Detailed annotations** (purpose + examples).
* A **unified cheat sheet** for study or quick reference.

---


