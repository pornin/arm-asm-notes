# Bit Twiddling

## Some Tricks with Multiplications

On ARMv7-M, the `umull`, `umlal` and `umaal` opcodes can be used to
efficiently perform several non-trivial operations; in particular, these
opcodes, contrary to most other 1-cycle instructions, can write to two
different registers.

If register `ra` is known to be zero, then two other registers `rb` and
`rc` can be cleared in a single cycle:

~~~
@ ra == 0
@ (rb, rc) <- (0, 0)
@ ra is preserved
.macro CLEAR_TWO  ra, rb, rc
        umull   \rb, \rc, \ra, \ra
.endm
~~~

Register `rx` can be decremented with saturation at zero, i.e. if the
register is already zero then it is not decremented. This requires another
register loaded with the constant -1 (preserved) and an extra scratch:

~~~
@ rm == -1
@ rx <- rx - 1 if rx != 0; rx is preserved if rx == 0
@ rm is preserved
@ rt is scratch
.macro DECREMENT_SATZERO  rx, rm
        umull   rt, rx, rx, rm
.endm
~~~

## Split Even/Odd Bits

Given input `x`, we want to separate its value into even-indexed bits
and odd-indexed bits, i.e. move bits around such that bits 0, 2, 4, 6...
go to positions 0 to 15 in the output, while bits 1, 3, 5, 7... go
to positions 16 to 31. The macro `BIT_SPLIT_32` performs this operation
in 14 cycles, provided that the `ASPR.GE` flags have been set to the
appropriate value beforehand:

~~~
@ On input, ASPR.GE must be 0110.
@ Split bits of rx into rd.
@ rx is consumed
@ ASPR.GE is preserved
.macro  BIT_SPLIT_32  rd, rx
        eor     \rd, \rx, \rx, lsr #1
        and     \rd, \rd, #0x22222222
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #1
        eor     \rd, \rx, \rx, lsr #2
        and     \rd, \rd, #0x0C0C0C0C
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #2
        eor     \rd, \rx, \rx, lsr #4
        and     \rd, \rd, #0x00F000F0
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #4
        rev     \rd, \rx
        sel     \rd, \rd, \rx
.endm
~~~

The `ASPR.GE` flags are used for the final step (last two instructions
in `BIT_SPLIT_32`), which swap the two "middle bytes" of the value in
`rx`. Setting `ASPR.GE` takes three cycles:

~~~
        movw    r0, #0xFF00
        movt    r0, #0x00FF
        uadd8   r0, r0, r0
~~~

If only a single bit split operation needs to be done, then it is
more efficient to do the final byte movement without the `ASPR.GE`
flags, with these four instructions instead of `rev` + `sel`:

~~~
        eor     \rd, \rx, \rx, lsr #8
        and     \rd, \rd, #0x0000FF00
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #8
~~~

This brings the cost of `BIT_SPLIT_32` to 16 cycles (instead of 14)
but does not need the 3-cycle initialization of `ASPR.GE`. If multiple
`BIT_SPLIT_32` must be performed, though, then the method with `rev` and
`sel` is faster since the `ASPR.GE` flags need to be set only once.

`BIT_SPLIT_64` extends `BIT_SPLIT_32` to a 64-bit input:

~~~
@ On input, ASPR.GE must be 0110.
@ Split bits of ra:rb into ra:rb (ra receives the even-indexed bits)
@ rt is scratch
@ ASPR.GE is preserved
.macro  BIT_SPLIT_64  ra, rb, rt
        BIT_SPLIT_32  \rt, \ra
        BIT_SPLIT_32  \ra, \rb
        pkhtb   \rb, \ra, \rt, asr #16
        pkhbt   \ra, \rt, \ra, lsl #16
.endm
~~~

This is typically used in an implementation of SHA-3/SHAKE; for
SHAKE256, 17 `BIT_SPLIT_64` operations must be performed.

The reverse operations (re-interleaving bits) can be done similarly:

~~~
@ On input, ASPR.GE must be 0110.
@ Interleave bits of rx into rd.
@ rx is consumed
@ ASPR.GE is preserved
.macro  BIT_MERGE_32  rd, rx
        rev     \rd, \rx
        sel     \rx, \rd, \rx
        eor     \rd, \rx, \rx, lsr #4
        and     \rd, \rd, #0x00F000F0
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #4
        eor     \rd, \rx, \rx, lsr #2
        and     \rd, \rd, #0x0C0C0C0C
        eor     \rx, \rx, \rd
        eor     \rx, \rx, \rd, lsl #2
        eor     \rd, \rx, \rx, lsr #1
        and     \rd, \rd, #0x22222222
        eor     \rx, \rx, \rd
        eor     \rd, \rx, \rd, lsl #1
.endm

@ On input, ASPR.GE must be 0110.
@ Interleave bits of ra:rb into ra:rb (ra provides the even-indexed bits)
@ rt is scratch
@ ASPR.GE is preserved
.macro  BIT_MERGE_64  ra, rb, rt
        pkhtb   \rt, \rb, \ra, asr #16
        pkhbt   \rb, \ra, \rb, lsl #16
        BIT_MERGE_32    \ra, \rb
        BIT_MERGE_32    \rb, \rt
.endm
~~~
