# Arithmetics

## Maximum (unsigned)

Given inputs `rx` and `ry`, output `rd` is set to the greater of the
two values (interpreted as unsigned integers).

### Compatible ARMv6-M

On ARMv6-M, the comparison result can be turned into a condition,
which can then be used in a selection sequence, for a total of 5 cycles:

~~~
@ rd <- max_u(rx, ry) (maximum of values, unsigned interpretation)
@ rx is preserved
@ ry is preserved
@ rt is scratch
.macro UMAX_1  rd, rx, ry, rt
        cmp     \rx, \ry
        sbcs    \rt, \rt
        subs    \rd, \rx, \ry
        bics    \rd, \rt
        adds    \rd, \ry
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, a combination of comparison and selection yields a 4-cycle
method:

~~~
@ rd <- max_u(rx, ry) (maximum of values, unsigned interpretation)
@ rx is preserved
@ ry is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx
.macro UMAX_2  rd, rx, ry, rt
        subs    \rt, \rx, \ry
        sbcs    \rt, \rt
        bics    \rd, \rx, \rt
        mls     \rd, \ry, \rt, \rd
.endm
~~~

However, we can do better, down to 3 cycles, if the output is written
in the same register as the input `rx`:

~~~
@ rx <- max_u(rx, ry) (maximum of values, unsigned interpretation)
@ ry is preserved
@ rt is scratch
@ ru is scratch
@ VARIANT: ru can be the same register as ry (consumed)
.macro UMAX_3  rx, ry, rt, ru
        subs    \ru, \ry, \rx
        sbcs    \rt, \rt
        umaal   \rx, \ru, \rt, \ru
.endm
~~~

## Minimum (unsigned)

Given inputs `rx` and `ry`, output `rd` is set to the lesser of the
two values (interpreted as unsigned integers).

### Compatible ARMv6-M

On ARMv6-M, the comparison result can be turned into a condition,
which can then be used in a selection sequence, for a total of 5 cycles:

~~~
@ rd <- min_u(rx, ry) (minimum of values, unsigned interpretation)
@ rx is preserved
@ ry is preserved
@ rt is scratch
.macro UMIN_1  rd, rx, ry, rt
        cmp     \ry, \rx
        sbcs    \rt, \rt
        subs    \rd, \rx, \ry
        bics    \rd, \rt
        adds    \rd, \ry
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, a combination of comparison and selection yields a 4-cycle
method:

~~~
@ rd <- min_u(rx, ry) (minimum of values, unsigned interpretation)
@ rx is preserved
@ ry is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx
.macro UMIN_2  rd, rx, ry, rt
        subs    \rt, \ry, \rx
        sbcs    \rt, \rt
        bics    \rd, \rx, \rt
        mls     \rd, \ry, \rt, \rd
.endm
~~~

By writing the output into the same register as the first operand (`rx`),
we can get a 3-cycle method:

~~~
@ rx <- min_u(rx, ry) (minimum of values, unsigned interpretation)
@ ry is preserved
@ rt is scratch
@ ru is scratch
@ VARIANT: ru can be the same register as ry (consumed)
.macro UMIN_3  rx, ry, rt, ru
        subs    \ru, \ry, \rx
        sbcs    \rt, \rt
        umlal   \ru, \rx, \rt, \ru
.endm
~~~

## Combined Maximum and Minimum (unsigned)

If we have two values `rx` and `ry`, then we can compute the minimum and
the maximum of the two (unsigned interpretation), faster than doing the
two computations separately, and also faster than doing a comparison
then using a conditional swap.

### Compatible ARMv6-M

A 5-cycle combined min/max is obtained, with outputs written over the
inputs, and two scratch registers:

~~~
@ (rx, ry) <- (min_u(rx, ry), max_u(rx, ry))
@ rt is scratch
@ ru is scratch
.macro UMINMAX_1  rx, ry, rt, ru
        subs    \rt, \ry, \rx
        sbcs    \ru, \ru
        ands    \rt, \ru
        subs    \ry, \rt
        adds    \rx, \rt
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, a 4-cycle method is the following (with two scratch registers):

~~~
@ (rx, ry) <- (min_u(rx, ry), max_u(rx, ry))
@ rt is scratch
@ ru is scratch
.macro UMINMAX_2  rx, ry, rt, ru
        subs    \rt, \ry, \rx
        sbcs    \ru, \ru
        umlal   \ry, \rx, \ru, \rt
        subs    \rx, \ru
.endm
~~~

## Generalized Adder

The `adcs` opcode is a generalized adder, accepting a carry on input,
and producing a carry on output. The carry uses the `C` flag. On
ARMv7-M, a one-cycle full adder can also be implemented with the
`umaal` opcode *without* using or impacting the `C` flag; this can be
used if two big integer addition chains must be interleaved:

~~~
@ rm = 1
@ rc \in { 0, 1 }
@ rx:rc <- rx + ry + rc  (addition with carry)
@ rm is preserved
.macro FULL_ADDER  rx, ry, rc
        umaal   rx, rc, rm, ry
.endm
~~~

This operation generalizes to adding `m*y` to `x` by setting `rm` to the
multipler `m` (the carry value in `rc` then extends to the range `[0,m]`).
In particular, by setting `m` to `2^n - 1` and using the same register
for `rx` and `ry`, then we obtain a method to left-shift a big integer
by `n` bits with an amortized cost of 1 cycle per word:

~~~
        @ Example: left-shift of r0:r1:r2:r3:r4:r5 by 7 bits; extra top
        @ bits go to r6.
        movs    r6, #0
        movs    r7, #127         @ 127 == 2^7 - 1
        umaal   r0, r6, r7, r0
        umaal   r1, r6, r7, r1
        umaal   r2, r6, r7, r2
        umaal   r3, r6, r7, r3
        umaal   r4, r6, r7, r4
        umaal   r5, r6, r7, r5
~~~

## 32x32->64 Multiplication (unsigned)

Given two 32-bit integers (considered unsigned), compute their product
over 64 bits.

### Compatible ARMv6-M

A 17-cycle multiplication routine can be obtained, with some usage
constraints: low word of the output overwrites the first operand,
high word of the output goes to a distinct register, second operand
is consumed, two scratch registers are needed:

~~~
@ rx:rh <- rx * ry  (32x32->64 unsigned multiplication)
@ ry is consumed
@ rt is scratch
@ ru is scratch
.macro UMUL32x32_64_1  rx, rh, ry, rt, ru
        lsrs    \rt, \ry, #16      @ rt <- hi(y)
        lsrs    \rh, \rx, #16
        muls    \rh, \rt           @ rh <- hi(x)*hi(y)

        uxth    \ry, \ry           @ ry <- lo(y)
        lsrs    \ru, \rx, #16      @ ru <- hi(x)
        uxth    \rx, \rx           @ rx <- lo(x)
        muls    \rt, \rx           @ rt <- lo(x)*hi(y)
        muls    \ru, \ry           @ ru <- hi(x)*lo(y)
        muls    \rx, \ry           @ rx <- lo(x)*lo(y)

        @ rx:rh <- rx:rh + rt*2^16; we use ry as scratch
        lsrs    \ry, \rt, #16
        lsls    \rt, \rt, #16
        adds    \rx, \rt
        adcs    \rh, \ry

        @ rx:rh <- rx:rh + ru*2^16; we use ry as scratch
        lsrs    \ry, \ru, #16
        lsls    \ru, \ru, #16
        adds    \rx, \ru
        adcs    \rh, \ry
.endm
~~~

### Specific to ARMv7-M

On ARMv7-M, the 32x32->64 unsigned multiplication is simply the
`umull` opcode.

## 64x64->64 Multiplication

Given two 64-bit integers, compute their product (64 low bits only).
Since we only keep the low 64 bits, the output is the same regardless of
whether we consider the inputs signed or unsigned.

### Compatible ARMv6-M

Given `rxl:rxh` and `ryl:ryh`, the product (64-bit) is written into
`rxl:rxh`; `ryl:ryh` is consumed, and an extra scratch register is
needed. The sequence cost is 21 cycles:

~~~
@ rxl:rxh <- rxl:rxh * ryl:ryh mod 2^64  (unsigned 64-bit mul, 64-bit output)
@ ryl is consumed
@ ryh is consumed
@ rt is scratch
.macro UMUL64x64_64_1  rxl, rxh, ryl, ryh, rt
        muls    \ryh, \rxl
        muls    \rxh, \ryl
        adds    \ryh, \rxh

        lsrs    \rt, \ryl, #16      @ rt <- hi(yl)
        lsrs    \rxh, \rxl, #16
        muls    \rxh, \rt           @ rxh <- hi(xl)*hi(yl)
        adds    \rxh, \ryh

        uxth    \ryl, \ryl          @ ryl <- lo(yl)
        lsrs    \ryh, \rxl, #16     @ ryh <- hi(xl)
        uxth    \rxl, \rxl          @ rxl <- lo(xl)
        muls    \rt, \rxl           @ rt <- lo(xl)*hi(yl)
        muls    \ryh, \ryl          @ ryh <- hi(xl)*lo(yl)
        muls    \rxl, \ryl          @ rxl <- lo(xl)*lo(yl)

        @ rxl:rxh <- rxl:rxh + rt*2^16; we use ryl as scratch
        lsrs    \ryl, \rt, #16
        lsls    \rt, \rt, #16
        adds    \rxl, \rt
        adcs    \rxh, \ryl

        @ rxl:rxh <- rxl:rxh + ryh*2^16; we use ryl as scratch
        lsrs    \ryl, \ryh, #16
        lsls    \ryh, \ryh, #16
        adds    \rxl, \ryh
        adcs    \rxh, \ryl
.endm
~~~

C compilers typically provide this routine as a stand-alone function,
to implement multiplication on 64-bit types (e.g. `uint64_t`). That
function can be implemented as follows, using the macro defined above:

~~~
        .align  2
        .global llmul
        .thumb
        .thumb_func
        .type   llmul, %function
llmul:
        mov     r12, r4
        UMUL64x64_64_1  r0, r1, r2, r3, r4
        mov     r4, r12
        bx      lr
        .size   llmul, .-llmul
~~~

Note that we use register `r12`, which is caller-saved, as temporary
storage for `r4`, which we need as scratch register (`r12` is a "high"
register and cannot be used for computations on ARMv6-M, but it is still
there and can be used for storage; using `r12` is faster than saving
`r4` on the stack). The overall function cost is 25 cycles (including
the final `bx lr`, but exclusing the `bl` call that is used to invoke
this function).

### Specific to ARMv7-M

A full 64-bit multiply can be done in 4 cycles on ARMv7-M:

~~~
@ rxl:rxh <- rxl:rxh * ryl:ryh mod 2^64  (64-bit mul with 64-bit output)
@ ryl is consumed
@ ryh is consumed
.macro UMUL64x64_64_2  rxl, rxh, ryl, ryh
        mul     \ryh, \rxl, \ryh
        mla     \rxh, \ryl, \rxh, \ryh
        umull   \rxl, \ryl, \rxl, \ryl
        add     \rxh, \rxh, \ryl
.endm
~~~

This sequence can be reduced to 3 cycles if we accept to obtain the
result in different registers:

~~~
@ rdl:rdh <- rxl:rxh * ryl:ryh mod 2^64  (64-bit mul with 64-bit output)
@ rxl is preserved
@ rxh is preserved
@ ryl is preserved
@ ryh is preserved
.macro UMUL64x64_64_3  rdl, rdh, rxl, rxh, ryl, ryh
        umull   \rdl, \rdh, \rxl, \ryl
        mla     \rdh, \rxl, \ryh, \rdh
        mla     \rdh, \rxh, \ryl, \rdh
.endm
~~~

## 64x64->128 Multiplication

On ARMv7-M, a full 64-bit multiplication with 128-bit output is obtained
in only 4 cycles:

~~~
@ rd0:rd1:rd2:rd3 <- rxl:rxh * ryl:ryh  (64x64->128 multiplication, unsigned)
@ VARIANT: rxl can be the same register as either rd0 or rd2
.macro UMUL64x64_128_1  rd0, rd1, rd2, rd3, rxl, rxh, ryl, ryh
        umull   \rd1, \rd3, \rxl, \ryh
        umull   \rd0, \rd2, \rxl, \ryl
        umaal   \rd1, \rd2, \rxh, \ryl
        umaal   \rd2, \rd3, \rxh, \ryh
.endm
~~~
