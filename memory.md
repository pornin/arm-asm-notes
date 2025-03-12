# Memory Access Notes

## Endianness

Nominally, the architecture supports little-endian and big-endian modes;
any single CPU that implements the architecture may use either mode, and
may optionally be able to support both with some implementation-defined
method to switch between the two.

In practice, most ARM Cortex-M0+ and M4 are used in little-endian mode,
and hand-written assembly commonly assumes little-endian.

## ARM Cortex-M0+

The ARM Cortex-M0+ supports 8-bit, 16-bit and 32-bit accesses. Each memory
access (read or write) is 2 cycles. All accesses must be aligned (i.e.
accesses to 16-bit slots must be at even addresses; accesses to 32-bit
slots must be at addresses multiple of 4).

The "load multiple" (`ldm`, `pop`) and "store multiple" (`stm`, `push`)
opcodes can read or write N words in N+1 cycles, which is better; they
are also size-efficient since a single 16-bit instruction can be used to
load or store multiple registers. In ARMv6-M, `ldm` and `stm`
necessarily increment the used index register (unless it is also used as
destination in `ldm`). `ldm` and `stm` can only load and store the low
registers (`r0` to `r7`).

`push` and `pop` use the stack pointer (register `r13`, also known as
`sp`). `push` can save `r14` (`lr`); `pop` can restore `r15` (`pc`). A
function that needs to call another function, or that needs `r14` as a
temporary register, will typically push `lr` on entry, and pop it back
into `pc` when exiting. If `lr` does not need to be saved, then it is
faster to not push it on entry, and to use `bx lr` to return.

Loads and store for a word (`ldr`, `str`) can use as index:

  * a general-purpose low register (`r0` to `r7`) with a small
    immediate non-negative offset (0 to 124 bytes, necessarily a multiple
    of 4);

  * the stack pointer (`sp`) or the program counter (`pc`) with a
    small non-negative offset (0 to 1020 bytes, necessarily a multiple
    of 4);

  * the sum of two arbitrary low registers.

Since the immediate offset is non-negative, PC-relative accesses used
for constants require that the loaded value is located after the code
that loads it, not before. The range is relatively small; thus, these
constants are usually immediately after the function that uses them.

For accesses to single bytes (`ldrb`, `strb`) and to halfwords (`ldrh`,
`strh`), addressing modes using `sp` or `pc` are not available. These
accesses are unsigned, i.e. loading a byte or halfword fills the upper
bits of the destination register with zeros. Sign-extending loads of
bytes (`ldrsb`) and of halfwords (`ldrsh`) are available, but they
support only the "sum of two low registers" syntax. Sign extension can
otherwise be achieved with extra opcodes (`sxtb`, `sxth`), but of course
with the corresponding overhead (1 extra instruction and 1 clock cycle).

## ARM Cortex-M4

The ARM Cortex-M4 supports all that the M0+ supports, but has also
more options:

  * Unaligned accesses (for a single-word or single-halfword) are
    tolerated. Penalty is 1 cycle for a misaligned halfword (16-bit
    chunk at an odd address). Penalty is 1 or 2 cycles for a misaligned
    word (32-bit word at address equal to 2 mod 4 is 1-cycle penalty;
    penalty is 2 cycles if the address is odd). Opcodes that load or
    store multiple words must still do only aligned accesses.

  * Addressing modes for `ldr`, `ldrb`, `ldrh`, `ldrsb` and `ldrsh`
    are extended and harmonized:

      * all registers can be used, not just low registers;

      * immediate offsets have a larger range and can be negative;

      * when an immediate value is added to an index register, that
        register value can be updated as well, either before (pre-indexed)
        or after (post-indexed) the access;

      * when two register values are added to compute the address, the
        second register can optionally be multiplied by 2, 4 or 8.

  * The `ldm`, `stm`, `push` and `pop` opcodes can use all registers,
    not just the low registers. The `ldmdb` and `stmdb` opcodes work
    like `ldm` and `stm` but use pre-decrement instead of post-increment
    for the index register (i.e. they load/store values at descending
    addresses).

  * The `ldrd` and `strd` can access two words at consecutive addresses,
    with arbitrary registers as destination or source (whereas `ldm`
    and `stm` use registers in increasing order only). These opcodes
    can use addressing with immediate offsets, including the variants
    that update the index register before or after the access. However,
    the "sum of two registers" addressing mode is not available for
    `ldrd` and `strd`.

Memory access timings are more complicated than in the M0+:

  * In general, a load (`ldr`) is 2 cycles, but successive loads will
    be pipelined, so that a sequence of N `ldr` opcodes will use 1+N
    cycles, provided that all loads are aligned and none of the loads
    uses for its address calculation the destination register of the
    immediately previous load.

  * A lone store (`str`) is 1 cycle, but if the next opcode performs
    a memory access then an extra 1-cycle penalty is applied. This is
    because the write uses an asynchronous buffer, which can operate
    in parallel with purely computational operations, but not with other
    memory accesses.

  * `ldrd` and `strd` are 3 cycles each. They do not pipeline with
    other instructions. If two words must be written at successive
    addresses, it is often preferable to use two `str` opcodes,
    separated by a computational instruction, so that the writes can
    be done with a cost of 2 cycles in total.

  * There can be contention with the instruction fetch unit. The exact
    conditions are not documented and may depend on how the CPU core is
    integrated by the microcontroller vendor. *Experimentally*, extra
    penalties appear in routines that perform memory accesses and in
    which instructions are not aligned. This means that in a
    time-critical routine which mixes computations and memory accesses,
    then all instructions that use a 32-bit encoding should appear only
    at addresses which are multiple of 4. In practice, this means:

      * the function should be 32-bit aligned (`.align 2` directive);

      * if instructions with 16-bit encoding are used, then they should
        be paired together;

      * use of explicit `.n` and `.w` instruction suffixes can help in
        ensuring that the instruction encoded size is as expected.
