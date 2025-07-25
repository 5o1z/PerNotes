## Overview

So the next level of userland pwn is kernel pwn. This is a very interesting topic, but also very hard to get into. The kernel is a very complex piece of system, and it is not easy to understand how it works. However, once you get the hang of it, it is a lot of fun to exploit (I think so). For most cases in userland, our goal is to exploit vulnerabilities remotely and finally get a remote shell, whether root or not. However, when pwning the kernel, in my opinion, the main goal is to escalate privilege from low-privilege user to root, or to escape from a sandbox (e.g. container), which could also be regarded as a special case of escalation of privilege. And these exploitations always happen locally, that is, on the victim machine.

## Table of Contents
1. [Kernel vs Userland: Similarities and Differences](#kernel-vs-userland-similarities-and-differences)
2. [Features of Kernel Exploit](#features-of-kernel-exploit)
    - [Attack Surface](#attack-surface)
    - [Sharing resources](#sharing-resources)
    - [Sharing Heap Space](#sharing-heap-space)
3. [Setup the environment](#setup-the-environment)
    - [Build a kernel for WSL](#build-a-kernel-for-wsl)
    - [QEMU](#qemu)
    - [Disk Image](#disk-image)
4. [Debugging the kernel with GDB](#debugging-the-kernel-with-gdb)
    - [Obtaining Root Privileges](#obtaining-root-privileges)
    - [Attach to QEMU](#attach-to-qemu)

## Kernel vs Userland: Similarities and Differences

The Linux kernel is a ELF binary as well, so kernel PWN and userland PWN share lots of common points. You can analyse binaries (e.g. IDA, objdump, ropr) with the same tools and exploit with similar techniques (e.g. ROP). By the way, instead of ROPgadget, I recommend ropr if you want to analyse vmlinux for ROP gadgets, in that ropr is faster. However, there are differences between kernel PWN and userland PWN technically:

- Different attack modes: For userland, when we exploit a vulnerability, we usually want to get a remote shell, which is a process running on the victim machine. However, when we exploit a kernel vulnerability, we usually want to escalate privilege or escape from a sandbox, which means we want to run code with higher privilege (e.g. root) or outside the sandbox.

- Different interaction with the system: In userland, there are many ways to interact with the user program. For example, you can send to and receive from the remote service to exploit. Another common way is to deliver a crafted malicious file to trigger vulnerabilities within local softwares like office suites. Ways to interact with kernel may be different: system calls, page faults, signals, pseudo-filesystems, device drivers and so on.

- Different mitigation mechanisms: The kernel has its own set of security mechanisms like KASLR, SMEP, SMAP, KPTI, and so on. These mechanisms are different from those in userland, and they may require different techniques to bypass. Howerver, some mitigations are same in both userland and kernel (e.g. ASLR, NX, Stack Canary).

- Different exploiting techniques. As we have discussed, some basic methodologies and techniques like stack overflow and ROP work in both cases, but to bypass different mitigations we need different techniques or tricks.

## Features of Kernel Exploit

### Attack Surface
The kernel has a large attack surface, which includes system calls, device drivers, and kernel modules. This means that there are many potential vulnerabilities that can be exploited. The kernel is also responsible for managing system resources, which means that a successful exploit can lead to a complete compromise of the system. Since the code in the kernel is running with high privilege, a successful exploit can lead to a complete compromise of the system called LPE (Local Privilege Escalation).

The other is a vulnerability in kernel modules such as device drivers. A device driver is an interface that makes it easy to interact with external devices (such as printers) primarily from user space. Device drivers must also be running with root privileges. So if there is a bug, it will lead to LPE.

### Sharing resources
Another feature of kernel exploits is that resources are shared. In userland, there was usually one process to attack, and by exploiting that process, attacks such as taking a shell were performed. On the other hand, programs such as the Linux kernel and device drivers are shared by all processes that use the operating system. Anyone can use system calls at any time, and you don't know who will operate the device driver and when. In other words, when you write code that runs in kernel space, you can easily embed vulnerabilities if you don't always program with the intention of multithreading. **This means that when you use potentially conflicting data, such as global variables, you have to take locks. Kernel space programming is hard.**

### Sharing Heap Space

In the Linux kernel, the heap is shared by all drivers and the kernel itself. This is different from user-space programs, where each program has its own separate heap. In user-space, even if there’s a heap overflow, whether it can be exploited depends on that specific program. But in the kernel, a single heap overflow in one driver can corrupt memory used by other drivers or the kernel, since they all share the same heap space. From an attacker's point of view, this shared heap has both pros and cons:

- **Pro:** Even a small heap vulnerability in one part of the kernel can potentially lead to privilege escalation (LPE). This is because the Linux kernel uses many objects with function pointers, and an attacker could overwrite one to hijack execution (e.g., control the instruction pointer, RIP).

- **Con:** The heap’s layout and state are much harder to predict. In user-space, the heap is more stable and controlled — especially in simple programs — so advanced heap exploitation techniques like “House of XXX” are possible. But in the kernel, the attacker can’t be sure what data is next to the overflowing chunk or what will happen after a Use-after-Free, since many components are using the same memory area unpredictably.

-> That's mean `Heap Spray` is important in kernel exploits


## Setup the environment

### Build a kernel for WSL

Let's say you want to build a kernel for WSL for your own use. You can follow the steps below:

1. **Install the necessary tools:**
   Make sure you have the required packages installed to build the kernel. You can install them using:
   ```bash
    sudo apt update && sudo apt install build-essential flex bison libssl-dev libelf-dev bc python3 pahole cpio
   ```

2. **Download the kernel source code:**
   Clone the WSL2-Linux-Kernel repository from [GitHub](https://github.com/microsoft/WSL2-Linux-Kernel). For example:
   ```bash
   git clone https://github.com/microsoft/WSL2-Linux-Kernel.git --depth=1 -b linux-msft-wsl-5.15.y
    ```
    In my case I have cloned the Kernel version 5.15, you can change the branch to the version you want.

3. **Configure the kernel:**
    Navigate to the kernel source directory and configure the kernel options. You can use the default WSL configuration as a starting point:
   ```bash
    cd WSL2-Linux-Kernel
    cp Microsoft/config-wsl .config
    ```
    If you copied the default WSL2 kernel config file to .config, you can now edit using your preferred text editor, such as `vim` or `nano`. You can also use `make menuconfig` to interactively configure the kernel options.

4. **Build the kernel:**
    After configuring the kernel, you can build it using:
    ```bash
    make -j$(nproc)
    ```
    However, if you use the config-wsl file instead of the `.config` one, you can run the following command to build the kernel:
    ```bash
    make KCONFIG_CONFIG=Microsoft/config-wsl -j4
    ```
    **Note:** The `-j4` option specifies the number of parallel jobs to run during the build process. You can adjust this number based on your CPU cores.

5. **Install the kernel:**
    Once the build is complete, you can install the kernel modules, and header.
    ```bash
    sudo make modules_install headers_install
    ```
    This will install the kernel modules and headers to the appropriate directories. So the last step is to copy the kernel image to the WSL directory:
    ```bash
    cp arch/x86/boot/bzImage <Somewhere you want to put the kernel image>
    ```

    Add it to the WSL configuration file `.wslconfig`:
    ```ini
    [wsl2]
    kernel=<Path to the kernel image>
    ```
6. **Restart WSL:**
    After modifying the `.wslconfig` file, you need to restart WSL for the changes to take effect. You can do this by running:
    ```bash
    wsl --shutdown
    ```
    Then, start WSL again, and it should use the new kernel you built.
7. **Verify the kernel version:**
    You can verify that WSL is using the new kernel by running:
    ```bash
    uname -r
    ```
    This should display the version of the kernel you just built.


### QEMU

QEMU (Quick Emulator) is a versatile emulator that can be used to run Linux kernels in a virtualized environment. It is particularly useful for testing and debugging kernel code without affecting your host system. To install QEMU, you can use the following command:
```bash
sudo apt install qemu-system-x86
```

### Disk Image
To run a kernel in QEMU, you need a disk image that contains the root filesystem. Disk images are generally created and distributed either as raw binaries of a file system such as ext or in a format called `cpio`. The `cpio` also grants permission information, so you need to properly assign the owner of the file to root when editing the file system. All of the above commands are run with root privileges, so there is no problem, but if you are troublesome, you can pack them with option: `--owner=root`

```bash
mkdir root
cd root; cpio -idv < ../rootfs.cpio
...
find . -print0 | cpio -o --format=newc --null [--owner=root] > ../rootfs_updated.cpio
```

## Debugging the kernel with GDB

### Obtaining Root Privileges

When debugging a kernel exploit at hand, it is often inconvenient with general user privileges. In particular, when setting breakpoints in the processing of the kernel or kernel drivers, or investigating what function the leaked address is, you cannot obtain kernel space address information without root privileges. When the kernel starts, one program name `init` is first executed. This program has different paths depending on the configuration, but in many cases `/init` or `/sbin/init`. So that program will look like this:
```bash
#!/bin/sh
# devtmpfs does not get automounted for initramfs
/bin/mount -t devtmpfs devtmpfs /dev

# use the /dev/console device node from devtmpfs if possible to not
# confuse glibc's ttyname_r().
# This may fail (E.G. booted with console=), and errors from exec will
# terminate the shell, so use a subshell for the test
if (exec 0</dev/console) 2>/dev/null; then
    exec 0</dev/console
    exec 1>/dev/console
    exec 2>/dev/console
fi

exec /sbin/init "$@"
```

There’s nothing particularly important written here, but it runs `/sbin/init`. In minimal environments such as those used in CTFs, it’s common for `/init` to directly install drivers or launch a shell. In fact, if you write `/bin/sh` before the final exec, you can launch a shell with root privileges at kernel boot. However, in that case, necessary initialization steps like driver installation won't be executed, so we won’t overwrite this file for now. From `/sbin/init`, the script `/etc/init.d/rcS` is eventually executed. This script runs all files in `/etc/init.d/` that start with the letter `S`. In among them, there is a file look like this:
```bash
#!/bin/sh

##
## Setup
##
mdev -s
mount -t proc none /proc
mkdir -p /dev/pts
mount -vt devpts -o gid=4,mode=620 none /dev/pts
chmod 666 /dev/ptmx
stty -opost
# echo 2 > /proc/sys/kernel/kptr_restrict
#echo 1 > /proc/sys/kernel/dmesg_restrict

##
## Install driver
##
insmod /root/vuln.ko
mknod -m 666 /dev/holstein c `grep holstein /proc/devices | awk '{print $1;}'` 0

##
## User shell
##
echo -e "\nBoot took $(cut -d' ' -f1 /proc/uptime) seconds\n"
echo "[ Holstein v1 (LK01) - Pawnyable ]"
setsid cttyhack setuidgid 1337 sh

##
## Cleanup
##
umount /proc
poweroff -d 0 -f
```

The line `setsid cttyhack setuidgid 1337 sh` launches a shell with the user ID 1337, which represents a non-root user. So to obtain root privileges, you can change the user ID to 0, which is the root user. However, this is not a good idea when you run the exploit in local, because you won't know your exploit will successfully gain root privileges or not. So just set it back to non-root user when you run the exploit, and set it to root when you debug the exploit.

```bash
setsid cttyhack setuidgid 0 sh
```

### Attach to QEMU

QEMU provides a way to debug the kernel using GDB. You can start QEMU with the option `-gdb tcp::1234` to enable GDB debugging. Then, in GDB terminal, you can connect to the QEMU instance using the command `target remote localhost:1234`. This allows you to set breakpoints, inspect memory, and step through the kernel code.

## <Updating ...>
