# Lab 01 report

- Student name: Nazar Pechkovskiy
- GitHub username: Raz1kson
- Image filename: am335x-debian-13.6-base-v6.12-armhf-2026-07-24-4gb.img.gz
- SHA-256 (paste the checksum and state whether it matches the lab): 796777685b309328abcab3a122f426282c83fbde7f481ebe3e20829441f878a9  am335x-debian-13.6-base-v6.12-armhf-2026-07-24-4gb.img.gz (matches the official hash)


Replace each placeholder with your own command output. Label HOST and BBB
results. Do not include passwords or private keys.

## Running Linux on BBB

### BBB: `uname -r`

```text
6.12.96-bone64
```

### BBB: `cat /etc/os-release`

```text
PRETTY_NAME="Debian GNU/Linux 13 (trixie)"
NAME="Debian GNU/Linux"
VERSION_ID="13"
VERSION="13 (trixie)"
VERSION_CODENAME=trixie
DEBIAN_VERSION_FULL=13.6
ID=debian
HOME_URL="https://www.debian.org/"
SUPPORT_URL="https://www.debian.org/support"
BUG_REPORT_URL="https://bugs.debian.org/"
```

### BBB: `lsblk`

```text
NAME         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
mmcblk0      179:0    0 14.5G  0 disk 
├─mmcblk0p1  179:1    0   36M  0 part /boot/firmware
├─mmcblk0p2  179:2    0  512M  0 part [SWAP]
└─mmcblk0p3  179:3    0 13.9G  0 part /
zram0        253:0    0  241M  0 disk [SWAP]
mmcblk1      179:256  0  3.6G  0 disk 
└─mmcblk1p1  179:257  0  3.6G  0 part 
mmcblk1boot0 179:512  0    2M  1 disk 
mmcblk1boot1 179:768  0    2M  1 disk 
```

### BBB: `ip addr`

```text
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host noprefixroute 
       valid_lft forever preferred_lft forever
2: eth0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc mq state DOWN group default qlen 1000
    link/ether 78:04:73:98:4d:62 brd ff:ff:ff:ff:ff:ff
    altname end0
    altname enx780473984d62
3: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 78:04:73:98:4d:65 brd ff:ff:ff:ff:ff:ff
    inet 192.168.7.2/30 brd 192.168.7.3 scope global usb0
       valid_lft forever preferred_lft forever
    inet6 fe80::7a04:73ff:fe98:4d65/64 scope link proto kernel_ll 
       valid_lft forever preferred_lft forever
```

## First user-space program

### HOST, in this lab directory: `file build/hello`

```text
build/hello: ELF 32-bit LSB pie executable, ARM, EABI5 version 1 (SYSV), dynamically linked, interpreter /lib/ld-linux-armhf.so.3, BuildID[sha1]=56082b83d1852f2c77fbd0e1898fe8a27a2235ca, for GNU/Linux 3.2.0, not stripped
```

### BBB: `/home/debian/labs/lab01/hello`

```text
Debian GNU/Linux 13

BeagleBoard.org Debian Trixie Base Image 2026-07-24
Support: https://bbb.io/debian
default username is [debian]

Hello from Lab 01 on BeagleBone Black!
```

## Shared kernel workspace

### HOST: `ls -lh ~/bbb-workspace/kernel/bb-kernel/deploy`

```text
total 35M
-rw-rw-r-- 1 vboxuser vboxuser 177K Sep 25 13:28 6.12.109-bone72-dtbs.tar.zst
-rw-rw-r-- 1 vboxuser vboxuser  26M Sep 25 13:28 6.12.109-bone72-modules.tar.zst
-rwxrwxr-x 1 vboxuser vboxuser 8.4M Sep 25 13:27 6.12.109-bone72.zImage
-rw-rw-r-- 1 vboxuser vboxuser 206K Sep 25 13:27 config-6.12.109-bone72
```

### HOST: `cat ~/bbb-workspace/kernel/bb-kernel/KERNEL/include/config/kernel.release`

```text
6.12.109-bone72
```

Do the full HOST and BBB kernel-release strings match exactly?

```text
Ні, рядки не збігаються. На хості скомпільовано 6.12.109-bone72, тоді як на платі працює 6.12.96-bone64.
```

Why must Lab 02 use a matching kernel tree?

```text
Для збірки зовнішніх модулів ядра потрібно мати точні заголовки того ядра, що запущене на платі.
```

Submit this completed report with `src/hello.c` and `Makefile`. Do not commit
images, executables, toolchains, kernel sources, or build artifacts.
