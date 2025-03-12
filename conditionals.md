# Conditionals

All "conditional" operations described here work with a condition value
provided in a register with value `-1` (`0xFFFFFFFF`) for "true", or `0`
(`0x00000000`) for "false". The macros assume that no other value is
used; if a condition register has a value distinct from either, then
what is obtained is unspecified.

The *carry flag* (`C`) can be converted into a condition value (`0` or
`-1`) in any register `rx` with `sbcs rx, rx` (this is compatible with
ARMv6-M). If `C` is cleared, then this sets `rx` to `-1`; otherwise
(`C` is set), `rx` is set to `0`. Note that `C` is set when an addition
operation triggers an output carry, or when a subtraction operation
does *not* trigger an output borrow (in ARM CPUs, the `C` flag operates
as the opposite of a borrow flag in subtraction; this is unlike x86
CPUs, where a borrow sets `C`).

## Conditional Move

A conditional move sets an output register `rx` to the value of another
register `ry` if a condition value `rc` is `-1`, or leaves `rx`
unchanged otherwise.

### Compatible ARMv6-M

A conditional move can be done in 3 cycles; the operand `ry` is consumed:

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == -1; rx preserved if rc == 0
@ ry is consumed
@ rc is preserved
.macro COND_MOVE_1  rx, ry, rc
        bics    \rx, \rc
        ands    \ry, \rc
        orrs    \rx, \ry
.endm
~~~

Another method also costs 3 cycles, and preserves `ry`, but it needs an
extra scratch register (if the same register as `ry` is used for the
scratch, then this is equivalent to the previous method).

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == -1; rx preserved if rc == 0
@ ry is preserved
@ rc is preserved
@ rt is scratch
@ VARIANT: rt and ry can be the same register (consumed)
.macro COND_MOVE_3  rx, ry, rc, rt
        subs    \rt, \rx, \ry
        ands    \rt, \rc
        subs    \rx, \rt
.endm
~~~

Both methods can also work with an inverted condition, i.e. do the
data move only if `rc` is `0`, and not do it if `rc` is `-1`:

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == 0; rx preserved if rc == -1
@ ry is consumed
@ rc is preserved
.macro CONDNOT_MOVE_1  rx, ry, rc
        ands    \rx, \rc
        bics    \ry, \rc
        orrs    \rx, \ry
.endm
~~~

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == 0; rx preserved if rc == -1
@ ry is preserved
@ rc is preserved
@ rt is scratch
@ VARIANT: rt and ry can be the same register (consumed)
.macro CONDNOT_MOVE_3  rx, ry, rc, rt
        subs    \rt, \rx, \ry
        bics    \rt, \rc
        subs    \rx, \rt
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, we can do the move in 2 cycles. Moreover, the alteration to
`ry` is predictable: `ry` is preserved if no move happens (`rc == 0`),
or cleared otherwise (`rc == -1`):

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == -1; rx preserved if rc == 0
@ ry <- 0 if rc == -1; ry preserved if rc == 0
@ rc is preserved
.macro COND_MOVE_4  rx, ry, rc
        bics    \rx, \rc            @ can be skipped if rx is known to be zero
        umlal   \ry, \rx, \rc, \ry
.endm
~~~

A variant preserves `ry` in all cases:

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == -1; rx preserved if rc == 0
@ ry is preserved
@ rc is preserved
.macro COND_MOVE_5  rx, ry, rc
        bics    \rx, \rc            @ can be skipped if rx is known to be zero
        mls     \rx, \ry, \rc, \rx
.endm
~~~

In both methods above, if `rx` is zero on input, then the first instruction
(the `bics` opcode) can be removed.

A 2-cycle move with an inverted condition consumes `ry`:

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == 0; rx preserved if rc == -1
@ ry is consumed
@ rc is preserved
.macro CONDNOT_MOVE_4  rx, ry, rc
        subs    \ry, \rx
        umaal   \rx, \ry, \rc, \ry
.endm
~~~

A 2-cycle move with an inverted condition and preserving `ry` can also be
done, but it requires an extra scratch register:

~~~
@ rc \in { 0, -1 }
@ rx <- ry if rc == 0; rx preserved if rc == -1
@ ry is preserved
@ rc is preserved
@ rt is scratch
@ VARIANT: rt and ry can be the same register (consumed)
.macro CONDNOT_MOVE_5  rx, ry, rc, rt
        subs    \rt, \ry, \rx
        umaal   \rx, \rt, \rc, \rt
.endm
~~~

## Selection

In a selection operation, an output register `rd` is set to the value
of either input register `rx` or input register `ry`, depending on
whether condition `rc` is `0` or `-1`, respectively.

### Compatible ARMv6-M

A 3-cycle method preserves all the inputs:

~~~
@ rc \in { 0, -1 }
@ rd <- rx if rc == 0; rd <- ry if rc == -1
@ rx is preserved
@ ry is preserved
@ rc is preserved
@ VARIANT: rd can be the same register as rx
.macro SELECT_1  rd, rx, ry, rc
        subs    \rd, \rx, \ry
        bics    \rd, \rc
        adds    \rd, \ry
.endm
~~~

In the method above, output `rd` can be the same register as input `rx`
(the input whose value is used for the output if `rc == 0`). An alternate
implementation allows instead `rd` to be the same register as input `ry`
(the input whose value is used for the output if `rc == -1`):

~~~
@ rc \in { 0, -1 }
@ rd <- rx if rc == 0; rd <- ry if rc == -1
@ rx is preserved
@ ry is preserved
@ rc is preserved
@ VARIANT: rd can be the same register as ry
.macro SELECT_2  rd, rx, ry, rc
        subs    \rd, \ry, \rx
        ands    \rd, \rc
        adds    \rd, \rx
.endm
~~~

Both methods can work with an inverted condition by simply exchanging
the `rx` and `ry` parameters.

### Specific to ARMv7-M

On ARMv7-M, a 2-cycle method for selection exists; it preserves all
inputs, and allows the output register to be the same register as
input `rx`:

~~~
@ rc \in { 0, -1 }
@ rd <- rx if rc == 0; rd <- ry if rc == -1
@ rx is preserved
@ ry is preserved
@ rc is preserved
@ VARIANT: rd can be the same register as rx
.macro SELECT_3  rd, rx, ry, rc
        bics    \rd, \rx, \rc
        mls     \rd, \ry, \rc, \rd
.endm
~~~

Another method uses the `ASPR.GE` flags, which are set by a few
instructions in the SIMD unit, and can be used for selection of values
with the `sel` instruction:

~~~
@ rc \in { 0, -1 }
@ rd <- rx if rc == 0; rd <- ry if rc == -1
@ rx is preserved
@ ry is preserved
@ rc is preserved
@ rt is scratch
@ VARIANT: rd may be the same register as rx, ry or rt. Moreover, rt
@ may be the same register as rc (which is then consumed).
.macro SELECT_4  rd, rx, ry, rc, rt
        uadd16  \rt, \rc, \rc
        sel     \rd, \ry, \rx
.endm
~~~

That method actually uses only bits 15 and 31 of `rc`. An extra scratch
register is used, but destination register `rd` can be the same register
as `rx`, `ry` or `rt` (`rt` must still be distinct from `rx` and `ry`).

Since the `ASPR.GE` flags are set by `uadd16` but not modified by `sel`,
the `SELECT_4` macro can be expanded to perform multiple selection
operations at a marginal cost of 1 cycle per word; for instance:

~~~
        @ Use condition r14 (0 or -1) to set r0:r1:r2:r3 to either
        @ r4:r5:r6:r7 (if r14 == -1) or r8:r10:r11:r12 (if r14 == 0).
        uadd16  r0, r14, r14
        sel     r0, r4, r8
        sel     r1, r5, r10
        sel     r2, r6, r11
        sel     r3, r7, r12
~~~

## Conditional Addition

In conditional addition, value `ry` is added to `rx` if condition `rc`
is true (`-1`); if the condition is false (`0`) then `rx` is unmodified.

### Compatible ARMv6-M

On ARMv6-M, this can be done in 2 cycles with the straightforward code:

~~~
@ rc \in { 0, -1 }
@ rx <- rx + ry if rc == -1; rx preserved if rc == 0
@ ry preserved if rc == -1; ry <- 0 if rc == 0
@ rc is preserved
.macro COND_ADD_1  rx, ry, rc
        ands    \ry, \rc
        adds    \rx, \ry
.endm
~~~

Note that operand `ry` is cleared if the addition is *not* done, but
preserved if the addition is performed.

### Specific to ARMv7-M

On ARMv7-M, the conditional addition can be done in a single cycle,
and preserving `ry`:

~~~
@ rc \in { 0, -1 }
@ rx <- rx + ry if rc == -1; rx preserved if rc == 0
@ ry is preserved
@ rc is preserved
.macro COND_ADD_2  rx, ry, rc
        mls     \rx, \ry, \rc, \rx
.endm
~~~

A variant clears `ry` when the addition is done, and preserves it when
the addition is not performed (this is distinct from the `COND_ADD_1`
macro semantics):

~~~
@ rc \in { 0, -1 }
@ rx <- rx + ry if rc == -1; rx preserved if rc == 0
@ ry <- 0 if rc == -1; ry preserved if rc == 0
@ rc is preserved
.macro COND_ADD_3  rx, ry, rc
        umlal   \ry, \rx, \rc, \ry
.endm
~~~

## Conditional Subtraction

Conditional subtraction is similar to conditional addition, except that
a subtraction is done instead of an addition.

### Compatible ARMv6-M

As in the case of conditional addition, a 2-cycle variant clears `ry`
when the subtraction is not done:

~~~
@ rc \in { 0, -1 }
@ rx <- rx - ry if rc == -1; rx preserved if rc == 0
@ ry preserved if rc == -1; ry <- 0 if rc == 0
@ rc is preserved
.macro COND_SUB_1  rx, ry, rc
        ands    \ry, \rc
        subs    \rx, \ry
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, a 1-cycle conditional subtraction preserves `ry`:

~~~
@ rc \in { 0, -1 }
@ rx <- rx - ry if rc == -1; rx preserved if rc == 0
@ ry is preserved
@ rc is preserved
.macro COND_SUB_2  rx, ry, rc
        mla     \rx, \ry, \rc, \rx
.endm
~~~

A variant consumes `ry` is `rc == -1`, which does not seem to provide
any advantage over `COND_SUB_2` (the actual replacement value is either
`2*ry` if `rx >= ry`, or `2*ry - 1` if `rx < ry`):

~~~
@ rc \in { 0, -1 }
@ rx <- rx - ry if rc == -1; rx preserved if rc == 0
@ ry consumed if rc == -1; ry preserved if rc == 0
@ rc is preserved
.macro COND_SUB_3  rx, ry, rc
        umlal   \rx, \ry, \rc, \ry
.endm
~~~

## Conditional Swap

A conditional swap exchanges the contents of registers `rx` and `ry`
if the condition `rc` is true (`-1`), but leaves them unchanged if
the condition is false (`0`).

### Compatible ARMv6-M

A 4-cycle conditional swap can be performed, using an extra scratch
register:

~~~
@ rc \in { 0, -1 }
@ (rx, ry) <- (ry, rx) if rc == -1; rx and ry preserved if rc == 0
@ rc is preserved
@ rt is scratch
.macro COND_SWAP_1  rx, ry, rc, rt
        subs    \rt, \rx, \ry
        ands    \rt, \rc
        subs    \rx, \rt
        adds    \ry, \rt
.endm
~~~

The classic method is often expressed with XORs. Here, we use additions
and subtractions because the ARMv6-M architecture offers a three-operand
subtraction opcode, which saves one cycle over the classic method using
`eors` (which only has a two-operand mode in ARMv6-M).

### Specific to ARMv7-M

On ARMv7-M, a conditional swap can be done in 3 cycles, still using one
extra register:

~~~
@ rc \in { 0, -1 }
@ (rx, ry) <- (ry, rx) if rc == -1; rx and ry preserved if rc == 0
@ rc is preserved
@ rt is scratch
.macro COND_SWAP_2  rx, ry, rc, rt
        subs    \rt, \rx, \ry
        mla     \rx, \rc, \rt, \rx
        umlal   \rt, \ry, \rc, \rt
.endm
~~~

A 2-cycle variant exists, but requires that `ry` is not greater than
`rx` (considering both values as unsigned); otherwise, if `ry > rx`
and `rc == -1` then the output `ry` will be off by 1:

~~~
@ rc \in { 0, -1 }
@ rx >= ry (unsigned comparison)
@ (rx, ry) <- (ry, rx) if rc == -1; rx and ry preserved if rc == 0
@ rc is preserved
@ rt is scratch
.macro COND_SWAP_3  rx, ry, rc, rt
        subs    \rt, \rx, \ry
        umlal   \rx, \ry, \rc, \rt
.endm
~~~
