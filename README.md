# ARM Assembly Tricks

This repository contains the description of various tricks and other
optimization notes that can be used to make assembly implementations
smaller and/or faster on small ARM embedded CPUs, targeting in
particular the ARM Cortex-M0+ and the ARM Cortex-M4.

These notes are envisioned in the context of cryptographic
implementations, so there is some emphasis on mathematical aspects and
on *constant-time* processing (making computations in such a way that no
private information can be inferred from timing measurements, including
computation time but also indirect timing-based effects such as cache
misses or stalls due to memory access contention).

* [Documentation links](doclinks.md)
* [Memory accesses](memory.md)
* [Conditionals](conditionals.md)
* [Arithmetics](arithmetics.md)
* [Modular integers](modular.md)
* [Bit twiddling](bits.md)

# Conventions

In all descriptions:

  * Each code snippet is provided as a macro in GAS (GNU assembly) syntax,
    which used register names provided as macro parameters.

  * In ARMv6-M code (for ARM Cortex-M0+), the registers should be chosen
    among the "low half" (`r0` to `r7`). On ARMv7-M (for ARM Cortex-M4),
    the "high" registers (`r8` to `r12`, and `r14`) can also be used
    (though `r9` could be reserved by the platform).

  * Unless specified explicitly, all registers provided as parameters
    shall be distinct.

  * The following terms are used to qualify the registers used as input:

      - *preserved*: the input register value is not modified

      - *consumed*: the register contents are set to an unspecified value

      - *scratch*: the register contents on input are ignored; an
        unspecifed value is written into the register

  * All ARMv6-M code also works on ARMv7-M, hence code that can run on
    an ARM Cortex-M0+ is said to be "compatible ARMv6-M". Variants
    specific to ARMv7-M are listed if they provide an advantage (usually
    in terms of speed) over the ARMv6-M code.

  * The notation `rx:ry` means that two 32-bit registers are considered as
    a single 64-bit value, with `rx` being the *low* half, and `ry` the
    *high* half. The notation extends to more than two words, e.g.
    `ra:rb:rc:rd` for a 128-bit integer (`ra` is least significant,
    `rd` is most significant).
