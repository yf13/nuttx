# NuttX on RiscV

Some personal feelings about "NuttX on RiscV".

## Overview

Some high level things in common:

- Open: both are open source technologies.
- Adaptive: both are highly customizable.
- Scalable: both begin with MCU and grow toward powerful chips.

## Closer view

RiscV has 4 priledge levels: Machine, Hyperviosr, Supervisor and User. Chips can choose to support:

- Only M-mode,
- Both M- and U- modes with help from PMP,
- M-, S- and U- modes with help from MMU.
- M-, H-, S- and U- modes with help from PMP, MMU and IOMMU etc.

NuttX has multiple build modes as well:

- The FLAT build mode operates in a flat memory space.
- The PROTECTED build mode operates with MPU support to have separate kernel - user memory spaces to provide basic OS and apps isolation.
- The KERNEL build mode operates with MMU support to have separate address spaces for kernel and each user task. thus apps are isolated from each other.

More specifically we can use NuttX on RiscV as below[^2]:

 | Build     |  M-mode   | S-mode | U-mode  |
 | --------- | --------- | ------ | ------- |
 | FLAT      |  Yes      | Yes    |         |
 | PROTECTED |  Kernel   | ---    | Apps    |
 | KERNEL    |  SBI[^1]  | Yes    | Apps    |

Though RiscV wants to be as sucessful as Linux, Linux doesn't run on many low-end RiscV chips yet. On the other side, NuttX runs on almost all types of RiscV chips, from simple MCUs to multi-core SMP chips with MMU.

## Evolving together

NuttX is following RiscV closely. For example, it is the first RTOS supportinng [the RV64ILP32 ABI](nuttx-rv64ilp32) to manage 64-bit AIoT chips in an efficient manner.

If your target system requires Fast booting, Tiny footprint, Tealtime scheduing and Standard compliant programming interface, then NuttX is a good choice. The combination of NuttX and RiscV can create a solid base for your AIoT stack.

## Notes

[^1]: NuttX can work with standard OpenSBI or a builtin NuttSBI.
[^2]: Someone are also doing hypervisor with NuttX, details are unclear yet.
