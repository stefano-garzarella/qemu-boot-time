# qemu-boot-time

This repository collects perf-script and patches to measure the boot time
of a Linux VM with QEMU. Using I/O writes, we can trace events in the firmware
and Linux kernel.

We extended the I/O port addresses and values defined in qboot
[https://github.com/bonzini/qboot/blob/master/benchmark.h] adding new trace
points to trace the kernel boot time.


## Patches

#### Linux
Apply the `patches/linux.patch` to your Linux kernel in order to trace kernel
events
```shell
git checkout -b benchmark
git am qemu-boot-time/patches/linux.patch
```
#### QEMU
TODO
#### SeaBIOS
Apply the `patches/seabios.patch` to your SeaBIOS in order to trace bios
events
```shell
git checkout -b benchmark
git am qemu-boot-time/patches/seabios.patch

make clean distclean
cp /path/to/qemu/roms/config.seabios-256k .config
make oldnoconfig
```
You can use `qemu-system-x86_64 -bios seabios/out/bios.bin` to use the SeaBIOS
image built.
#### qboot
qboot already defines trace points, we just need to compile it defining
`BENCHMARK_HACK`

```shell
$ BIOS_CFLAGS="-DBENCHMARK_HACK=1" make
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
kill $PERF_PID

# Get the measurements
perf script -s qemu-boot-time/perf-script/qemu-perf-script.py -i $PERF_DATA

```


## Trace points
* QEMU
  * `qemu_init_end`: first kvm_entry (i.e. QEMU initialized has finished)
* Firmware (SeaBIOS + optionrom or qboot)
  * `fw_start`: first entry of the firmware
  * `fw_do_boot`: after the firmware initialization (e.g. PCI setup, etc.)
  * `linux_start_boot`: before the jump to the Linux kernel
  * `linux_start_pvhboot`: before the jump to the Linux PVH kernel
* Linux Kernel
  * `linux_start_kernel`: first entry of the Linux kernel
  * `linux_start_user`: before starting the init process

Trace points are printed only if they are recorded, so you can only enable
few of them.


## Example of output
```shell
$ perf script -s qemu-boot-time/perf-script/qemu-perf-script.py -i $PERF_DATA

in trace_begin
sched__sched_process_exec     1 55061.435418353   289738 qemu-system-x86      
kvm__kvm_entry           1 55061.466887708   289741 qemu-system-x86      
kvm__kvm_pio             1 55061.467070650   289741 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=1

kvm__kvm_pio             1 55061.475818073   289741 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=4

kvm__kvm_pio             1 55061.477168037   289741 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=3

kvm__kvm_pio             1 55061.558779540   289741 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=5

kvm__kvm_pio             1 55061.686849663   289741 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=6

sched__sched_process_exec     4 55067.461869075   289793 qemu-system-x86      
kvm__kvm_entry           4 55067.496402472   289796 qemu-system-x86      
kvm__kvm_pio             4 55067.496555385   289796 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=1

kvm__kvm_pio             4 55067.505067184   289796 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=4

kvm__kvm_pio             4 55067.506395502   289796 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=3

kvm__kvm_pio             4 55067.584029910   289796 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=5

kvm__kvm_pio             4 55067.704751791   289796 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=6

sched__sched_process_exec     0 55070.073823767   289827 qemu-system-x86      
kvm__kvm_entry           0 55070.110507211   289830 qemu-system-x86      
kvm__kvm_pio             0 55070.110694645   289830 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=1

kvm__kvm_pio             1 55070.120092692   289830 qemu-system-x86      rw=1, port=0xf5, size=1, count=1, val=4

kvm__kvm_pio             1 55070.121437922   289830 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=3

kvm__kvm_pio             1 55070.198628779   289830 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=5

kvm__kvm_pio             1 55070.315734630   289830 qemu-system-x86      rw=1, port=0xf4, size=1, count=1, val=6

in trace_end
Trace qemu-system-x86
1) pid 289738
 qemu_init_end: 31.469355
 fw_start: 31.652297 (+0.182942)
 fw_do_boot: 40.39972 (+8.747423)
 linux_start_boot: 41.749684 (+1.349964)
 linux_start_kernel: 123.361187 (+81.611503)
 linux_start_user: 251.43131 (+128.070123)
2) pid 289793
 qemu_init_end: 34.533397
 fw_start: 34.68631 (+0.152913)
 fw_do_boot: 43.198109 (+8.511799)
 linux_start_boot: 44.526427 (+1.328318)
 linux_start_kernel: 122.160835 (+77.634408)
 linux_start_user: 242.882716 (+120.721881)
3) pid 289827
 qemu_init_end: 36.683444
 fw_start: 36.870878 (+0.187434)
 fw_do_boot: 46.268925 (+9.398047)
 linux_start_boot: 47.614155 (+1.34523)
 linux_start_kernel: 124.805012 (+77.190857)
 linux_start_user: 241.910863 (+117.105851)

Avg
 qemu_init_end: 34.228732
 fw_start: 34.403161 (+0.174429)
 fw_do_boot: 43.288918 (+8.885757)
 linux_start_boot: 44.630088 (+1.34117)
 linux_start_kernel: 123.442344 (+78.812256)
 linux_start_user: 245.408296 (+121.965952)

Min
 qemu_init_end: 31.469355
 fw_start: 31.652297 (+0.182942)
 fw_do_boot: 40.39972 (+8.747423)
 linux_start_boot: 41.749684 (+1.349964)
 linux_start_kernel: 122.160835 (+80.411151)
 linux_start_user: 241.910863 (+119.750028)

Max
 qemu_init_end: 36.683444
 fw_start: 36.870878 (+0.187434)
 fw_do_boot: 46.268925 (+9.398047)
 linux_start_boot: 47.614155 (+1.34523)
 linux_start_kernel: 124.805012 (+77.190857)
 linux_start_user: 242.882716 (+118.077704)
```
