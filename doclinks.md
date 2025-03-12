# Documentation

## Instruction Set Architecture

The ISA specifies all instructions, how they are encoded, and what
they compute.

  * ARMv6-M: [ARM DDI 0419E](https://developer.arm.com/documentation/ddi0419/latest/) (detailed instruction list in section A6.7)

  * ARMv7-M: [ARM DDI O403E.e](https://developer.arm.com/documentation/ddi0403/latest/) (detailed instruction list in section A7.7)

## Processor Core Manuals

The manual of a processor core describes how a specific core implements
an ISA. This is where information about instruction timings is found.

  * ARM Cortex-M0+: [Cortex-M0+ Technical Reference Manual (revision r0p1)](https://developer.arm.com/documentation/ddi0484/latest/) (instruction timings in section 3.3)

  * ARM Cortex-M4: [Arm® Cortex®-M4 Processor Technical Reference Manual (revision r0p1)](https://developer.arm.com/documentation/100166/0001/) (instruction timings in section 3.3) (instruction timings in section 3.3)

Note: in the older r0p0 revision of the M4 reference manual, the `mls`
and `mla` instructions were listed as taking 2 cycles, while in the
real CPU they take only 1 cycle each. Revision r0p1 is correct.

## Application Binary Interface

The calling convention describes rules for function callers and callees,
such as register allocation and stack alignment.

  * Old standard (ATPCS): [The ARM-Thumb Procedure Call Standard Specification](https://developer.arm.com/documentation/espc0002/latest/)

  * New standard (AAPCS): [Procedure Call Standard for the Arm® Architecture (2024Q3)](https://github.com/ARM-software/abi-aa/releases/download/2024Q3/aapcs32.pdf)

ABI releases are maintained on GitHub: [https://github.com/ARM-software/abi-aa/releases](https://github.com/ARM-software/abi-aa/releases)

ATPCS and AAPCS are very similar, but they differ for handling of
floating-point arguments. For low-level cryptographic routines, the most
important points are:

  * First four (32-bit) arguments from in registers `r0` to `r3`.

  * Output (32-bit) is in `r0` (for a 64-bit integer output, `r0`
    is the low word, `r1` the high word).

  * Registers `r4` to `r11` are callee-saved; if a routine modifies them,
    then it must put back the original value on exit.

  * Register `r9` is nominally reserved. It is typically used for handling
    position-independent code, or thread-local storage. In some (rare)
    platforms it must be maintained at all times (not just
    saved-and-restored) to support asynchronous access (signals); in
    most others, its original value should be restored when calling an
    external function, or returning to an external caller. In
    micro-controllers, it is usually available as callee-saved. Portable
    assembly should refrain from using `r9`.

  * Register `r12` is free. It may be used by thunks added by the linker;
    thus, it is not (necessarily) preserved across calls or returns.

  * Register `r14` (also called `lr`) receives the return address. It can be
    used for computations, but the returned address must have been saved
    somewhere beforehand.

  * Stack should in general be 64-bit aligned across calls. Externally
    callable functions receive a 64-bit aligned stack; when calling an
    external function, the stack alignment should be preserved.

## Microcontroller Datasheet

The microcontroller vendor assembles a CPU core with RAM, ROM (Flash),
peripheral support, and other elements. In particular, some caches, DMA
support, and interconnection matrix are part of these elements, and may
influence operation timings.

  * [STM32F407](https://www.st.com/resource/en/datasheet/stm32f405zg.pdf)
    (the STM32F407 "discovery" board is one of the classic test boards among
    cryptographers for benchmarking code on an ARM Cortex M4).
