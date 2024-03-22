# Linux-ck-tt

[![PKGBUILD CI](https://github.com/RiverOnVenus/linux-ck-tt/actions/workflows/build.yml/badge.svg)](https://github.com/RiverOnVenus/linux-ck-tt/actions/workflows/build.yml)

The Linux-ck-tt kernel and modules with [Con Kolivas](https://github.com/ckolivas)' hrtimer patches and Task Type CPU scheduler  by [Hamad Al Marri](https://github.com/hamadmarri) and with some other patches. Built on the [Linux-ck](https://aur.archlinux.org/packages/linux-ck/) maintained by [graysky](https://github.com/graysky2).

- [Con Kolivas' hrtimer patches](https://github.com/ckolivas/linux/tree/5.12-ck) and the recommended 1000 Hz tick rate. 
- [TT-CPU-Scheduler](https://github.com/hamadmarri/TT-CPU-Scheduler) - The goal of the Task Type (TT) scheduler is to detect tasks types based on their behaviours and control the schedulling based on their types. ## **Temporarily not supported after version 6.6. Hamad is re-implementing TT from scratch on top of Baby scheduler.**
- [kernel_compiler_patch](https://github.com/graysky2/kernel_compiler_patch) enables compiler optimizations for additional CPUs.
- [BBR v3](https://github.com/google/bbr/blob/v3/README.md) is a congestion control algorithm proposed by Google.
- [clearlinux](https://github.com/clearlinux-pkgs/linux) from Intel's Clear Linux project. Provides performance and security optimizations.
- [CJKTTY](https://github.com/zhmars/cjktty-patches) supports displaying CJK Unified Ideographs on Linux tty.

# Build and install

Open `/etc/pacman.conf` and comment out the original Architecture, then add the new Architecture.

```
#Architecture = auto
Architecture = x86_64 x86_64_v3
```

Selecting the correct CPU optimized package.

```
/lib/ld-linux-x86-64.so.2 --help | grep supported
```

If `x86-64-v3 (supported, searched)` is in the output, use the *Generic-x86-64-v3 (GENERIC_CPU3)*.

You can compile it yourself and choose the optimization option that suits you.

```
git clone https://github.com/RiverOnVenus/linux-ck-tt.git

cd linux-ck-tt/linux-ck-tt

updpkgsums && makepkg -srci
```

You can also [download](https://github.com/RiverOnVenus/linux-ck-tt/releases) the compiled package.

# Clang and DKMS

~~When you use a kernel compiled by CLANG/LLVM/LTO, some modules that use DKMS need to be recompiled with CLANG/LLVM. Otherwise DKMS will fail.~~

~~You need to modify the `/etc/dkms/framework.conf` file, add two lines to the end of the file: `export LLVM=1`, `export CC=clang`.~~

~~If you have done that, just reinstall or install the kernel compiled with CLANG/LLVM/LTO and DKMS will not fail again.~~

Dkms([v3.0.2](https://github.com/dell/dkms/releases/tag/v3.0.2)) support for Clang, use the latest version of dkms and you'll be fine.

# Check if TT CPU Scheduler is enabled

This start-up message should appear in the kernel ring buffer when TT in enabled, use:

```
# dmesg | grep -i 'TT CPU'
```

You can see: `TT CPU scheduler v6.0 by Hamad Al Marri.`

# TT balancer option

`kernel.sched_tt_balancer_opt` can be set to 4 values:

- 0: Normal TT balancer
- 1: Candidate Balancer (which is an addition to normal TT balancer -  good for reponsiveness (perfomance gets affected when #CPUs > 4))
- 2: CFS balancer (default - good for perfomance/throughput)
- 3: Power save balancer (tries its best to avoid running tasks on idle cpus - saves power)

You can change the balancer option at run time. [See more](https://github.com/hamadmarri/TT-CPU-Scheduler/blob/master/README.md)

# Sysctl configuration improving performance

```
# See https://wiki.archlinux.org/title/Sysctl for more information.

# Networking
# Increasing this value for high speed cards may help prevent losing packets:
net.core.netdev_max_backlog = 16384

# The upper limit on how many connections the kernel will accept (default 128):
net.core.somaxconn = 8192

# The default the Linux network stack is not configured for high speed large file 	  transfer across WAN links (i.e. handle more network packets) and setting the correct values may save memory resources:
net.core.rmem_default = 1048576
net.core.rmem_max = 16777216
net.core.wmem_default = 1048576
net.core.wmem_max = 16777216
net.core.optmem_max = 65536
net.ipv4.tcp_rmem = 4096 1048576 2097152
net.ipv4.tcp_wmem = 4096 65536 16777216

# It is also possible increase the default 4096 UDP limits:
net.ipv4.udp_rmem_min = 8192
net.ipv4.udp_wmem_min = 8192

# Enable TCP Fast Open
net.ipv4.tcp_fastopen = 3

# Tweak the pending connection handling
net.ipv4.tcp_max_syn_backlog = 8192
net.ipv4.tcp_max_tw_buckets = 2000000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 10
net.ipv4.tcp_slow_start_after_idle = 0

# Enable MTU probing
net.ipv4.tcp_mtu_probing = 1

# Change TCP keepalive parameters
net.ipv4.tcp_keepalive_time = 120
net.ipv4.tcp_keepalive_intvl = 10
net.ipv4.tcp_keepalive_probes = 6

# Enable CAKE
net.core.default_qdisc = cake

# Virtual memory
# vm.dirty_background_ratio default 10
# vm.dirty_ratio default 20
# For 16GB RAM
vm.dirty_background_ratio = 2
vm.dirty_ratio = 4
# And
# For 8GB RAM
# vm.dirty_background_ratio = 4
# vm.dirty_ratio = 8

# VFS cache
# Decreasing the virtual file system (VFS) cache parameter value 
# may improve system responsiveness (default 100)
# vm.vfs_cache_pressure = 50

# Swappiness (default 60)
# For Solid State Drives
# vm.swappiness = 100
# See https://chrisdown.name/2018/01/02/in-defence-of-swap.html
# Or
# For zram
# vm.swappiness = 100


```

# Changing I/O scheduler if you want

```
# cat /sys/block/sda/queue/scheduler
```

To change the active I/O scheduler to *bfq* for device *sda*, use:

```
# echo bfq > /sys/block/sda/queue/scheduler
```

Or create file `/etc/udev/rules.d/60-ioschedulers.rules`:

```
# set scheduler for NVMe
ACTION=="add|change", KERNEL=="nvme[0-9]n[0-9]", ATTR{queue/scheduler}="bfq"
# set scheduler for SSD and eMMC
ACTION=="add|change", KERNEL=="sd[a-z]|mmcblk[0-9]*", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="bfq"
# set scheduler for rotating disks
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
```

**Reboot or force [udev#Loading new rules](https://wiki.archlinux.org/title/Udev#Loading_new_rules)**:

If rules fail to reload automatically, use:

```
# udevadm control --reload
```

To manually force *udev* to trigger your rules, use:

```
# udevadm trigger
```
