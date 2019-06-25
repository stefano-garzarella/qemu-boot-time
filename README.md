# qemu-boot-time

This repository collects perf-script and patches to measure the boot time
of a Linux VM with QEMU. Using I/O writes, we can trace events in the firmware
and Linux kernel.

We extended the IO port addresses and values defined in qboot
[https://github.com/bonzini/qboot/blob/master/benchmark.h] adding new trace
points to trace the kernel boot time.


## Patches

#### Linux
Apply the `patches/linux.patch` to your kernel in order to trace Linux kernel
events
```shell
git checkout -b benchmark
git am qemu-boot-time/patches/linux.patch
```
#### QEMU
TODO
#### SeaBIOS
TODO
#### qboot
qboot already defines trace points, we just need to compile it defining
`BENCHMARK_HACK`

```shell
$ BENCHMARK_HACK=1 make
```


## Prerequisites
The following steps allow `perf record` to get the kvm trace events:

```shell
echo 1 > /sys/kernel/debug/tracing/events/kvm/enable
echo -1 > /proc/sys/kernel/perf_event_paranoid
mount -o remount,mode=755 /sys/kernel/debug
mount -o remount,mode=755 /sys/kernel/debug/tracing
```


## How to use

```shell
# Start perf record to get the trace events
PERF_DATA="qemu_perf.data"
perf record -a -e kvm:kvm_entry -e kvm:kvm_pio -e sched:sched_process_exec \
-o $PERF_DATA &
PERF_PID=$!

# You can run QEMU multiple times to get also some statistics (Avg/Min/Max)
qemu-system-x86_64 -machine q35,accel=kvm -bios qboot/bios.bin \
                   -kernel linux/bzImage ...
qemu-system-x86_64 -machine q35,accel=kvm -bios qboot/bios.bin \
                   -kernel linux/bzImage ...
qemu-system-x86_64 -machine q35,accel=kvm -bios qboot/bios.bin \
                   -kernel linux/bzImage ...

# Stop perf record
$ kill $PERF_PID

# Get the measurements
$ perf script -s qemu-boot-time/perf-script/qemu-perf-script.py -i $PERF_DATA
```


## Trace points
* QEMU
  * `qemu_init_end`: first kvm_entry (i.e. QEMU initialized has finished)
* Firmware (SeaBIOS + optionrom or qboot)
  * `fw_start`: first entry of the firmware
  * `fw_do_boot`: after the firmware initialization (e.g. PCI setup, etc.)
  * `linux_start_boot`: before the jump to the Linux kernel
* Linux Kernel
  * `linux_start_kernel`: first entry of the Linux kernel
  * `linux_start_user`: before starting the init process

Trace points are printed only if they are recorded, so you can only enable
few of them.
