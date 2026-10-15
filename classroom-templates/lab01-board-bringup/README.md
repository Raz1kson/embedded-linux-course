# Lab 01 — BeagleBone Black Bring-Up and Kernel Workspace

Complete this lab before starting any kernel-module labs.

In this lab you will prepare the BeagleBone Black, verify that Linux is running, cross-compile a first ARM user-space program, and create the shared kernel workspace used by Lab 02 and later labs.

Work in `classroom-templates/lab01-board-bringup/` on branch `lab01`
in your personal course fork. Before starting, follow the
[student workflow](../../docs/student-workflow.md) to set up instructor review.

## Command labels

- **[HOST]** — your Linux PC.
- **[BBB]** — the BeagleBone Black.
- Replace placeholders such as `<SD_DEVICE>` and `<USB_INTERFACE>` with real values.
- Use `hostname`, `pwd`, and `uname -m` if you are unsure where you are.

## 1. Goals

By the end of this lab you will be able to:

- write a Debian image to microSD and boot the BBB from it;
- use a 3.3 V serial console;
- configure USB networking and SSH;
- inspect the running Linux system;
- cross-compile and run an ARM user-space program;
- create a local kernel workspace for later kernel-module labs.

The kernel workspace created here is required by Lab 02.

## 2. Repository structure

This lab directory contains:

```text
lab01-board-bringup/
├── README.md
├── Makefile
├── report.md
├── .gitignore
└── src/
    └── hello.c
```

Create the shared workspace outside your course fork on **[HOST]**:

```bash
mkdir -p ~/bbb-workspace/downloads
mkdir -p ~/bbb-workspace/kernel
mkdir -p ~/bbb-workspace/target-info
```

All kernel-module labs will use:

```text
~/bbb-workspace/kernel/bb-kernel/KERNEL
```

## 3. Required hardware

- BeagleBone Black;
- microSD card, 8 GB or larger, and a card reader;
- USB cable and Linux PC/VM;
- recommended: 3.3 V USB-to-TTL serial adapter.

Never connect a 5 V UART signal to the BBB serial pins.

## 4. Install HOST tools

**[HOST]**

```bash
sudo apt update
sudo apt install -y \
  git build-essential bc bison flex libssl-dev libncurses-dev \
  libelf-dev dwarves lzop u-boot-tools rsync xz-utils zstd \
  device-tree-compiler minicom picocom screen net-tools iproute2 \
  openssh-client gcc-arm-linux-gnueabihf g++-arm-linux-gnueabihf
```

Check the important tools:

```bash
arm-linux-gnueabihf-gcc --version
git --version
minicom --version
```

## 5. Download exactly this image

Open the official page:

<https://www.beagleboard.org/distros/beaglebone-black-debian-13-6-2026-07-24-iot-v6-12-x>

Select:

```text
BeagleBone Black Debian 13.6 2026-07-24 IoT (v6.12.x)
```

The downloaded filename must be:

```text
am335x-debian-13.6-base-v6.12-armhf-2026-07-24-4gb.img.xz
```

Why this one:

- `am335x` is the BeagleBone Black processor family;
- `IoT`/`base` is non-graphical and appropriate for command-line labs;
- `armhf` is the 32-bit ARM userspace used by BBB;
- `v6.12` matches the `am33x-v6.12` kernel-build branch used by this course.

Do **not** select an Xfce desktop image, an `AM57xx`/AI image, a v6.18 image,
or a v5.10-ti image. Do not select an eMMC flasher unless the instructor
specifically asks you to overwrite the onboard eMMC.

Save the image in `~/bbb-workspace/downloads`.

Verify the download on **[HOST]**:

```bash
cd ~/bbb-workspace/downloads
sha256sum am335x-debian-13.6-base-v6.12-armhf-2026-07-24-4gb.img.xz
```

Expected SHA-256:

```text
796777685b309328abcab3a122f426282c83fbde7f481ebe3e20829441f878a9
```

If it differs, do not flash the file; download it again.

## 6. Identify the microSD device

1. Remove the microSD card.
2. Run `lsblk`.
3. Insert the card and run it again.

**[HOST]**

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,MOUNTPOINTS,MODEL
```

The new disk is the card. Example only:

```text
sda      58G disk Transcend
├─sda1
├─sda2
└─sda3
```

Here the whole device is `/dev/sda`; `/dev/sda1` is only a partition.

> `dd` destroys all data on its output device. Never select the disk containing
> `/` or `/home`. If unsure, stop and show `lsblk` to the instructor.

In the commands below, replace `<SD_DEVICE>` with the verified whole device,
for example `/dev/sda`—not `/dev/sda1`.

## 7. Flash the image

Unmount any card partitions using their real names:

```bash
sudo umount /dev/sda1 2>/dev/null || true
sudo umount /dev/sda2 2>/dev/null || true
sudo umount /dev/sda3 2>/dev/null || true
```

Recheck the device, then write the image:

```bash
cd ~/bbb-workspace/downloads
xzcat am335x-debian-13.6-base-v6.12-armhf-2026-07-24-4gb.img.xz | \
  sudo dd of=<SD_DEVICE> bs=4M status=progress conv=fsync
sync
```

Meaning:

```text
xzcat           decompresses the .xz image
dd              writes the raw image
of=             selects the output disk
bs=4M           uses 4 MiB blocks
status=progress shows progress
conv=fsync      waits for data to reach the device
sync            flushes remaining cached writes
```

Wait for the shell prompt before removing the card.

## 8. Verify partitions and set the password

Remove and reinsert the card, then run:

```bash
lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,UUID,MOUNTPOINTS,MODEL
```

The image contains a small FAT32 boot partition and a Linux root filesystem;
exact sizes may vary.

The official image requires a password in `sysconf.txt` before first boot.
Mount the FAT32 partition if the desktop did not mount it automatically:

```bash
sudo mkdir -p /mnt/bbb_boot
sudo mount /dev/sda1 /mnt/bbb_boot
sudo nano /mnt/bbb_boot/sysconf.txt
```

Use the real partition name. Set:

```text
user_name=debian
user_password=your_temporary_password
```

Do not enable root login. Save, then safely unmount:

```bash
sync
sudo umount /mnt/bbb_boot
```

## 9. Boot from microSD

1. Power the BBB off completely.
2. Insert the card.
3. Hold the **BOOT/S2** button.
4. Apply power.
5. Hold BOOT/S2 for about 10–15 seconds, then release it.
6. Wait for Linux to boot.

## 10. Connect the serial console

Connect GND, adapter TX to BBB RX, and adapter RX to BBB TX. Use:

```text
115200 baud, 8 data bits, no parity, 1 stop bit, no flow control
```

**[HOST]**

```bash
dmesg | tail -30
ls /dev/ttyUSB*
sudo minicom -D /dev/ttyUSB0 -b 115200
```

If necessary, run `sudo minicom -s` and set both hardware and software flow
control to `No`. Exit minicom with `Ctrl-A`, then `X`.

## 11. Configure USB networking and SSH

The usual addresses are BBB `192.168.7.2` and HOST `192.168.7.1`. Find the HOST
USB interface; its generated name may look like `enx7804...`:

```bash
ip addr
```

If it has no IPv4 address:

```bash
sudo ip addr add 192.168.7.1/30 dev <USB_INTERFACE>
sudo ip link set <USB_INTERFACE> up
```

Test and connect:

```bash
ping -c 4 192.168.7.2
ssh debian@192.168.7.2
```

After reflashing, SSH may report a changed host key. First confirm that this is
your freshly flashed board, then remove only its old entry:

```bash
ssh-keygen -f "$HOME/.ssh/known_hosts" -R 192.168.7.2
ssh debian@192.168.7.2
```

## 12. Verify the running system

Run all of these on **[BBB]** and save the output for the report:

```bash
uname -a
uname -r
cat /etc/os-release
cat /proc/cpuinfo | head -40
lsblk
ip addr
ls /boot
find /boot -name '*.dtb' | head
```

Record the exact `uname -r`; do not assume it from the image name. When booting
from microSD, `mmcblk0` is commonly the card and `mmcblk1` the eMMC, but verify
using the mount point for `/` rather than relying only on numbering.

Run `exit` to return to the HOST shell.

## 13. Cross-compile and run the first program

The lab directory contains `src/hello.c`. In this first lab, run every build and
deployment command manually so that each step is visible.

**[HOST]**, open a terminal in this lab directory (the directory containing
this README and Makefile):

```bash
pwd
mkdir -p build
arm-linux-gnueabihf-gcc -O2 -Wall -Wextra src/hello.c -o build/hello
file build/hello
```

Expected `file` keywords: `ELF 32-bit`, `ARM`, `EABI5`. `x86-64` means the wrong
compiler was used.

Deploy and run:

```bash
ssh debian@192.168.7.2 'mkdir -p /home/debian/labs/lab01'
scp build/hello debian@192.168.7.2:/home/debian/labs/lab01/hello
ssh debian@192.168.7.2 'chmod +x /home/debian/labs/lab01/hello'
ssh debian@192.168.7.2 '/home/debian/labs/lab01/hello'
```

Expected output:

```text
Hello from Lab 01 on BeagleBone Black!
```

To inspect or test the program interactively instead, connect to the board:

```bash
ssh debian@192.168.7.2
cd /home/debian/labs/lab01
./hello
```

To rebuild from scratch on **[HOST]**, remove the generated file yourself and
repeat the compile command:

```bash
rm -f build/hello
arm-linux-gnueabihf-gcc -O2 -Wall -Wextra src/hello.c -o build/hello
```

## 14. Obtain and build the kernel tree

On **[HOST]**, obtain and build the kernel outside your course fork:

```bash
cd ~/bbb-workspace/kernel
git clone https://github.com/RobertCNelson/bb-kernel.git
cd bb-kernel
git checkout origin/am33x-v6.12 -b am33x-v6.12-course
./build_kernel.sh
```

This creates `~/bbb-workspace/kernel/bb-kernel` using the course branch
`am33x-v6.12`. The Linux source tree stays outside the course repository and its
forks and is required by Lab 02. On a new PC, repeat this lab to recreate
the workspace; do not copy the kernel tree into your course fork.

`bb-kernel` downloads Linux sources and its matching compiler. On the first
configuration screen, change nothing: choose `Exit`, then `Save: Yes`. The first
build takes a long time and several gigabytes.

Check the result:

```bash
ls -lh ~/bbb-workspace/kernel/bb-kernel/deploy
cat ~/bbb-workspace/kernel/bb-kernel/KERNEL/include/config/kernel.release
```

Typical artifacts include `zImage`, config, modules archive, and DTBs archive.
The prepared tree needed by Lab 02 is:

```text
~/bbb-workspace/kernel/bb-kernel/KERNEL
```

### Mandatory version check

The branch name `v6.12` is not a complete kernel release. Before Lab 02 compare:

```bash
# HOST
cat ~/bbb-workspace/kernel/bb-kernel/KERNEL/include/config/kernel.release

# BBB
uname -r
```

The complete releases must match. If they do not, do not load a module. Ask the
instructor for the course-pinned `bb-kernel` commit matching the image. Updating
an image or branch independently can change the exact release.

## 15. Common problems

### Initramfs says `/dev/mmcblk0p3 does not exist`

Check `ls /dev/mmc*`, `cat /proc/cmdline`, and `blkid`. Power off, reseat the
card, hold BOOT/S2 longer, reflash, try another card, and use stable 5 V power.

### USB interface exists but ping fails

Run `ip addr` on both machines. Verify BBB `192.168.7.2`, HOST `192.168.7.1`,
the correct HOST `enx...` interface, and link state.

### Exact `linux-headers-$(uname -r)` package is unavailable

This is normal for some BeagleBoard images. Use the matching HOST kernel tree;
never use headers from a different release.

### Cross-compiler is missing

```bash
sudo apt install gcc-arm-linux-gnueabihf
arm-linux-gnueabihf-gcc --version
```

## 16. Submission

Submit a PR from `lab01` to `master` **in your own fork**, following the
[student workflow](../../docs/student-workflow.md). Include the files and
report evidence listed below, and send the instructor the PR URL. If a
classroom platform or LMS is used, submit that URL there as well.

The lab is accepted only after `@ant112342` approves the PR. Address review
comments on the same branch and merge only after approval of the current work.

- `src/hello.c` and `Makefile` from this lab directory;
- completed `report.md` with your name, GitHub username, and the evidence below;
- image filename and SHA-256;
- `uname -r`, OS information, `lsblk`, and BBB IP address;
- `file build/hello` and program output on BBB;
- file list from `~/bbb-workspace/kernel/bb-kernel/deploy`;
- both full kernel-release strings from the mandatory version check;
- a short explanation of why Lab 02 needs a matching kernel tree.

Do not commit images, executables, toolchains, or kernel build output.

## 17. Success checklist

```text
[-] Correct official image downloaded and checksum verified
[-] BBB boots from microSD
[-] Serial console works
[-] USB networking and SSH work
[-] ARM user-space program runs on BBB
[-] Kernel tree builds on HOST
[-] HOST and BBB full kernel releases match before Lab 02
```
