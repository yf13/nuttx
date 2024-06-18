# NuttX with RV64ILP32

This page shares information about NuttX built with the `rv64ilp32` toolchain.

## The Problem

Many AIoT chips using RISC-V in the market take rv64 profiles but with limited memory size support. They can run 64-bit Linux but not too much room left to apps. The industry addressed this with the `rv64ilp32` ABI which uses 32-bit pointers on rv64 CPU to reduce footprint and improve cache performance, while at same time enjoying the 64-bit processing power, see the [LWN page](https://lwn.net/Articles/951187) for more background information.

Though the requirements for `rv64ilp32` from Linux looks more urgent than RTOS, but chasing of efficiency is also the key for RTOS.

So how can NuttX benefit from this `rv64ilp32` thing?

## The toolchain

After downloading the [RuyiSDK rv64ilp32 toolchain](https://github.com/ruyisdk/riscv-gnu-toolchain-rv64ilp32/releases), playing the `hello` demo and raising questions to the friendly toolchain experts, concepts about how it works have been established.

Here are the major points:

- The `rv64ilp32` program is of ELF-32 class, but with flag 0x25;
- The `-rsicv64ilp32` named QEMU tools in the toolchain not only support the `rv64ilp32` program format, but also support masking of the higher 32-bit addresses used by the program;
- The toolchain uses `-march=rv64`, `-mabi=ilp32` for rv64ilp32 binaries;
- The compiler has `__riscv_xlen_t==64` and `sizeof(void*)==32` defined for programs to adapt themselves;
- Access to addresses beyond 4G space shall be done via assembly as compiled address has been limited in 32-bit.

## Building NuttX

With toolchain knowledge and [some tweaks](https://github.com/apache/nuttx/pull/12475) like below:

- Add `rv64ilp32` toolchain in `arch/risc-v` Kconfig and Toolchain.cmake to support generation of nuttx-rv64ilp32 binary.
- Add a type for register width data in `arch/risc-v` layer, with some tweaking of the code base to support both existing 64-bit and 32-bit ABIs and the new rv64ilp32 ABI.

Thanks to NuttX's well designed structure, the `nuttx-ilp32` image can boot smoothly on the QEMU comes with the toolchain and `ostest` also passed, though there are still some cleanups left behind.

## QEMU results

Here are initial comparisons between normal `lp64` and the `rv64ilp32`, the baseline configuration is `rv-virt/nsh64`. The toolchain used for `lp64` is the stock `gcc-riscv64-unknown-elf` package version 10.2.0, and the rv64ilp32 toolchain used is the 2024.05.23 release downloaded from URL above.

Compiler switches `-g -Os` are both present, as this isn't a formal comparison but just to give the basic idea.

Static footprints with `C` extension:

```
$ size nuttx-*
   text    data     bss     dec     hex filename
 176861     397   10256  187514   2dc7a nuttx-ilp32
 173171     681   12416  186268   2d79c nuttx-lp64
```

Dynamic footprints with `C` extension:

```
nsh> cat /proc/version
NuttX version 12.4.0 77abbe5a43-dirty Jun  6 2024 16:22:56 rv-virt/nsh64ilp32
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33364764       8068   33356696       8052   33356664     21      2

nsh> cat /proc/version
NuttX version 12.4.0 5ce64a46a5-dirty Jun  6 2024 16:23:33 rv-virt/nsh64
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   33366008      10344   33355664      10312   33355616     21      2
```

We can see that the code size if `rv64ilp32` is slightly(2.13%) bigger, and data memory reduced 18% or 22% for static or dynamic regions.

### The C extension

When sharing the initial comparison with RuyiSDK people, they suggested to turn off the compressed instruction extension `C` and recheck the text sizes.

So here we have a another list, while the rv64ilp32 program's text size is smaller:

```
$ size nuttx-*
   text     data      bss      dec      hex  filename
 231321      397    10256   241974    3b136  nuttx-ilp32
 231441      681    12416   244538    3bb3a  nuttx-lp64
```

Experts further said that current RISC-V ISA C extension doesn't have enough considerations for the `rv64ilp32` ABI, thus code size is slightly larger when the ISA C extension is turned on.

Let's hope the RISC-V ISA C extension can evolve to support `rv64ilp32` better in the future.


## K230 results

With a few more tweaks ending with [this patch](https://github.com/apache/nuttx/pull/12528), NuttX FLAT build can run on CanMV230:

```
Hit any key to stop autoboot:  0
starting USB...
Bus usb-otg@91540000: dwc2_usb usb-otg@91540000: Core Release: 4.30a
USB DWC2
scanning bus usb-otg@91540000 for devices... 2 USB Device(s) found
       scanning usb for storage devices... 0 Storage Device(s) found
Waiting for Ethernet connection... done.
Using r8152_eth device
TFTP from server 10.0.1.1; our IP address is 10.0.1.2
Filename 'master.bin'.
Load address: 0x8000000
Loading: #############
         1.7 MiB/s
done
Bytes transferred = 187236 (2db64 hex)
## Starting application at 0x08000000 ...
ABC
NuttShell (NSH) NuttX-12.4.0
nsh> cat /proc/version
NuttX version 12.4.0 caa94ef64b-dirty Jun 19 2024 06:55:46 canmv230/nsh32
nsh> free
                 total       used       free    maxused    maxfree  nused  nfree
      Umem:   16574332       6396   16567936       6752   16567904     19      2
nsh> ps
  PID GROUP PRI POLICY   TYPE    NPX STATE    EVENT     SIGMASK           STACK  USED  FILLED COMMAND
    0     0   0 FIFO     Kthread   - Ready              0000000000000000 002032000816  40.1%  Idle_Task
    1     1 100 RR       Task      - Running            0000000000000000 003024002096  69.3%  nsh_main
nsh>
```
The data memory reducation is in line with that of QEMU. Performances of a few checked apps like [getprime](https://github.com/apache/nuttx-apps/tree/master/testing/getprime) and slightly tweaked [smallpt](https://www.kevinbeason.com/smallpt/) are no worse than that of the `rv64lp64` build, and one app [calib_udelay](https://github.com/apache/nuttx-apps/blob/master/examples/calib_udelay/calib_udelay_main.c) has 15% faster result.

Note that this isn't a formal test due to limited set of apps at hand, but the result is basically in line with [those published results](https://mp.weixin.qq.com/s/rsF7kriYYaiWQYmHGYGjlg).
