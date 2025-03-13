# Modular Arithmetics

We consider here computations modulo a given odd integer `p` (the modulus).
The modulus needs not be prime, but it must be odd.

The normal method is to use Montgomery multiplication. Let `R` be a
suitable power of two; in the methods below, we'll use `R = 2^32`. We
also consider the integer `m = -1/p mod 2^32` (in the `[0, 2^32-1]`
range): that integer is well-defined since `p` is odd, and thus
invertible modulo `2^32`. `m` can be computed dynamically with the
following C code (which uses
[Hensel's lifting lemma](https://en.wikipedia.org/wiki/Hensel%27s_lemma)):

~~~c
// Given odd p, return -1/p mod 2^32.
uint32_t
make_m(uint32_t p)
{
    uint32_t r = (p - (p << 2)) ^ 2;
    for (int i = 0; i < 3; i ++) {
        r *= 2 + r * p;
    }
    return r;
}
~~~

This function is rarely required to be optimized, but here is an assembly
routine that does it in 9 cycles (on ARMv7-M):

~~~
@ rp = odd integer
@ rm <- -1/rp mod 2^32
@ rp is preserved
@ rt is scratch
@ ru is scratch
@ rv is scratch
@ VARIANT: rm can be the same register as rp, rt, ru or rv
.macro NINV32  rm, rp, rt, ru, rv
        sub     \rt, \rp, \rp, lsl #2
        movs    \ru, #2
        eors    \rt, \ru
        mla     \rv, \rt, \rp, \ru
        muls    \rt, \rv
        mla     \rv, \rt, \rp, \ru
        muls    \rt, \rv
        mla     \rv, \rt, \rp, \ru
        mul     \rm, \rt, \rv
.endm
~~~

The *Montgomery representation* of an integer `x` is `x*R mod p`,
represented in an appropriate range. The *Montgomery reduction* of
an integer `z` returns `z/R mod p`.

*Montgomery multiplication* of `a` and `b` is Montgomery reduction
applied to `a*b`, i.e. `a*b/R mod p`. If Montgomery multiplication is
applied to the Montgomery representations of `x` and `y`, then we
get `(x*R)*(y*R)/R = (x*y)/R mod p`: the Montgomery multiplication
is a modular multiplication that works over Montgomery representation
of values. Converting from Montgomery to normal representation is
done with a Montgomery reduction; converting from normal to Montgomery
representation is done by doing a Montgomery multiplication with
`R^2 mod p` (a precomputed constant).

The Montogmery reduction of integer `z` uses the integer `m = -1/p mod R`:

 1. Compute `f = z*m mod R`

 2. Compute `y = z + f*p`. We can see that `y` and `z` are equal
    modulo `p`, but also that `y = 0 mod R`.

 3. Return `x = y/R`. Since `y` is a multiple of `R`, the division
    is exact over the integers. By choosing `R` to be a power of two,
    the division is a simple right-shift.

If `p < R` and `z < R`, then `y <= R - 1 + R*p - p < R*(p + 1)`.
After the division by `R`, we necessarily get a value `x` in `[0, p]`, i.e.
*almost* entirely reduced (we can still get `p` instead of `0` in some
cases). If `z` is greater, but `z < p*R`, and `p < R`, then we still
get that `x < 2*p`, which can be brought down to `[0, p-1]` with a
single conditional subtraction of `p`.

## Small Modulus

When the modulus fits on 16 bits, values can also fit on 16 bits each.
Depending on the used architecture, the optimal representation of integers
modulo `p` varies.

### On ARMv6-M

ARMv6-M offers a 32-bit multiplication opcode (`muls`) which is very
efficient on the ARM Cortex-M0+ (1 cycle), but returns only the low
32 bits of the output. There is no opcode to obtain the high 32 bits.
We can still have an efficient Montgomery reduction, but it has a
somewhat limited range.

We normalize integers modulo `p` into the range `[1, p]`: value zero is
presented by `p`, not by `1`. The Montgomery reduction of an integer `x`
can be done in 5 cycles (method is from
[eprint 2020/009](https://eprint.iacr.org/2020/009), section 2.4):

~~~
@ rp = modulus p (odd integer, lower than 2^16)
@ rm = -1/p mod 2^32
@ On input: 1 <= rx < 2^32 + 2^16 - (2^16 - 1)*p
@ rx <- rx / 2^32 mod p  (normalized into range [1, p])
@ rp is preserved
@ rm is preserved
.macro MONTYRED_1  rx, rp, rm
        muls    \rx, \rm
        lsrs    \rx, #16
        muls    \rx, \rp
        lsrs    \rx, #16
        adds    \rx, #1
.endm
~~~

Note that the 5-cycle cost assumes that `rp` and `rm` are already loaded
with the right constants; but they are preserved by the computation.

The output is already normalized in `[1, p]` and does not need any
additional processing. The input range is somewhat smaller than the full
32-bit range. For computing Montgomery multiplications, we need
products (over the integers) to fit in the allowed range: for all integers
`a` and `b`, both in `[1, p]`, the product `a*b` must be less than
`2^32 + 2^16 - (2^16 - 1)*p`. This requires that `p <= 40503`.

When `p` is smaller, the input range may be large enough to mutualize
reductions; for instance, if `p = 12289` (a common case in some
cryptographic schemes, e.g. Falcon/FN-DSA), then the allowed range is
`[1, 3489673217]`, which can process all values up to `23*p^2`: we can
thus compute a sum of up to 23 products of values modulo p, and do a
single reduction on the result.

Since `MONTYRED_1` cannot process correctly an input of `0`, operations
must take care to provide an output in `[1, p]`. A modular addition
that follows these rules can be done in 5 cycles:

~~~
@ rp = modulus p (odd integer, lower than 2^16)
@ rx \in [1, p]
@ ry \in [1, p]
@ rd <- (rx + ry) mod rp  (in [1, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx or ry
.macro ADDMOD_1  rd, rx, ry, rp, rt
        adds    \rd, \rx, \ry
        cmp     \rp, \rd
        sbcs    \rt, \rt
        ands    \rt, \rp
        subs    \rd, \rt
.endm
~~~

Modular subtraction can also be done in 5 cycles:

~~~
@ rp = modulus p (odd integer, lower than 2^16)
@ rx \in [1, p]
@ ry \in [1, p]
@ rd <- (rx - ry) mod rp  (in [1, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx or ry
.macro SUBMOD_1  rd, rx, ry, rp, rt
        subs    \rd, \ry, \rx
        sbcs    \rt, \rt
        ands    \rt, \rp
        adds    \rd, \rt
        subs    \rd, \rp, \rd
.endm
~~~

Halving modulo p is done in 5 cycles. This method actually works for
larger (odd) moduli too, up to `2^31`:

~~~
@ rp = modulus (odd), less than 2^31
@ rx \in [0, p]
@ rd <- rx/2 mod rp
@ If rx != 0 then rd != 0.
@ If rx != p then rd != p.
@ rx is preserved
@ rp is preserved
.macro HALFMOD_1  rd, rx, rp
        lsls    \rd, \rx, #31
        asrs    \rd, \rd, #31
        ands    \rd, \rp
        adds    \rd, \rx
        lsrs    \rd, \rd, #1
.endm
~~~

This halving preserves the range, i.e. if input is in `[1, p]` then output
is also in `[1, p]`, and if input is in `[0, p-1]`, then output is also
in `[0, p-1]`.

### On ARMv7-M

On ARMv7-M, we have can use `umaal` to perform an efficient Montgomery
reduction that supports the entire 32-bit range as input:

~~~
@ rp = modulus p (odd integer, lower than 2^16)
@ rm = -1/p mod 2^32
@ rx <- rx / 2^32 mod p  (into range [0, p])
@ rp is preserved
@ rm is preserved
@ rt is scratch
.macro MONTYRED_2  rx, rp, rm, rt
        mul     \rt, \rx, \rm
        umaal   \rm, \rx, \rt, \rp
.endm
~~~

This 2-cycle operation returns a value in `[0, p]`. Contrary to the
ARMv6-M code above, we do not have to take care to avoid zero. Moreover,
its large range means that this method is usable for all odd moduli up
to `0xFFFF` (`2^16 - 1`).

Addition modulo `p` (with `[0, p]` range) can be done in 4 cycles:

~~~
@ rp = modulus p (odd integer, lower than 2^31)
@ rx \in [0, p]
@ ry \in [0, p]
@ rd <- (rx + ry) mod rp  (in [0, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx or ry
@ VARIANT: if rx and/or ry is in [0, p-1], then the output is in [0, p-1].
.macro ADDMOD_2  rd, rx, ry, rp, rt
        adds    \rd, \rx, \ry
        subs    \rd, \rd, \rp
        and     \rt, \rp, \rd, asr #31
        adds    \rd, \rd, \rt
.endm
~~~

Subtraction modulo `p` (with `[0, p]` range) is only 3 cycles:

~~~
@ rp = modulus p (odd integer, lower than 2^31)
@ rx \in [0, p]
@ ry \in [0, p]
@ rd <- (rx - ry) mod rp  (in [0, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx or ry
@ VARIANT: if rx is in [0, p-1], then the output is in [0, p-1].
.macro SUBMOD_2  rd, rx, ry, rp, rt
        subs    \rd, \rx, \ry
        and     \rt, \rp, \rd, asr #31
        adds    \rd, \rd, \rt
.endm
~~~

Halving is also 3 cycles, and (as in the ARMv6-M version) works on
odd moduli up to `2^31`:

~~~
@ rp = modulus (odd), less than 2^31
@ rx \in [0, p]
@ rd <- rx/2 mod rp
@ If rx != 0 then rd != 0.
@ If rx != p then rd != p.
@ rx is preserved
@ rp is preserved
.macro HALFMOD_2  rd, rx, rp
        ubfx    \rd, \rx, #0, #1
        mla     \rd, \rd, \rp, \rx
        lsrs    \rd, \rd, #1
.endm
~~~

**If `p <= 0x3FFF` (i.e. `p` fits on 14 bits)** then we can further
optimize things with the SIMD unit:

~~~
@ rp = modulus p (odd integer, lower than 2^14)
@ rx \in [0, p]
@ ry \in [0, p]
@ rd <- (rx + ry) mod rp  (in [0, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx or ry
@ VARIANT: rp, rx and ry may hold two values (low and high halves) to
@ make two computations in parallel
.macro ADDMOD_3  rd, rx, ry, rp, rt
        sadd16  \rd, \rx, \ry
        ssub16  \rt, \rd, \rp
        sel     \rd, \rt, \rd
.endm
~~~

The `ADDMOD_3` macro runs in 3 cycles, which is better than the 4-cycle
method of `ADDMOD_2`, but it can support only smaller moduli, since the
`ssub16` opcode expects signed 16-bit values and sets `ASPR.GE`
correctly only if the addition result (from `sadd16`) is properly
non-negative in signed interpretation, i.e. fits on 15 bits.

There is a similar 3-cycle method for modular subtraction, with the
same constraint on `p` (at most 14 bits), and requiring an extra
constant of value zero (which is preserved):

~~~
@ rp = modulus p (odd integer, lower than 2^14)
@ rx \in [0, p]
@ ry \in [0, p]
@ rz = 0
@ rd <- (rx - ry) mod rp  (in [0, p])
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rt is scratch
@ rz is preserved
@ VARIANT: rd can be the same register as rx or ry
@ VARIANT: rp, rx and ry may hold two values (low and high halves) to
@ make two computations in parallel
.macro SUBMOD_3  rd, rx, ry, rp, rt, rz
        ssub16  \rd, \rx, \ry
        sadd16  \rt, \rd, \rp
        sel     \rd, \rt, \rd
.endm
~~~

By itself, `SUBMOD_3` does not seem to be better than `SUBMOD_2`;
however, like `ADDMOD_3`, `SUBMOD_3` supports **parallel execution**,
which `SUBMOD_2` does not. Indeed, `ADDMOD_3` and `SUBMOD_3` both
perform two modular operations in parallel, one using the low half
of each register (`rx`, `ry`, `rp`, `rd`) and the other using the
high half. In a computation that needs to perform multiple modular
operations in parallel (as is typical when working with polynomials),
then this naturally leads to an efficient and parallel storage and
computation:

  * Values are stored in 16 bits each.

  * Data is read by 32-bit words; each 32-bit word contains two values.

  * Modular additions and subtractions use `ADDMOD_3` and `SUBMOD_3`
    (with `rp` set to `p + p*2^16`), with two simultaneous operations.

  * For multiplications, the `smulbb`, `smulbt`, `smultb` and `smultt`
    opcodes are used to compute the multiplications with the correct
    halves of the input registers (thus, there is no overhead for
    extracting the 16-bit value from the 32-bit word), then using
    `MONTYRED_2` (the Montgomery reduction is not parallel, though). In
    some rare cases, where a sum of two products is needed, then `smuad`
    or `smuadx` can be used to get that sum (unreduced) in a single
    cycle (`smusd` and `smusdx` can potentially be used as well for a
    difference of products, but with an extra addition of a multiple of
    `p` to get a non-negative input to `MONTYRED_2`).

Parallel halving is also possible in 3 cycles:

~~~
@ rp = modulus (odd integer, lower than 2^15)
@ rxx = xl + xh*2^16, xl \in [0, p], xh \in [0, p]
@ rdd <- (xl/2 mod p) + (xh/2 mod p)*2^16
.macro HALFMODPP_1  rdd, rxx, rp
        and     \rdd, \rxx, #0x00010001
        mla     \rdd, \rdd, \rp, \rxx
        lsrs    \rdd, \rdd, #1
.endm
~~~

Note that `HALFMODPP_1` requires that both halves use the same modulus;
`ADDMOD_3` and `SUBMOD_3` can work with two distinct moduli for the low
and high halves.

## Medium-Size Modulus

When the modulus is larger, the tricks described above no longer work.
If `2^16 < p < 2^31` then partial Montgomery reduction (still using
`R = 2^32`) can be done in 3 cycles:

~~~
@ rp = modulus p (odd integer, lower than 2^31)
@ rm = -1/p mod 2^32
@ rx and ry such that rx*ry <= p + (p - 1)*2^32
@ rd <- rx*ry/2^32 mod p  into range [0, 2*p-1]
@ rx is preserved
@ ry is preserved
@ rp is preserved
@ rm is preserved
@ rt is scratch
@ ru is scratch
@ VARIANT: rx and ry can be the same register (Montgomery squaring).
@ rd, rt and/or ru can use the same registers as rx and/or ry, as long
@ as they are distinct from each other.
.macro MONTYMUL_1  rd, rx, ry, rp, rm, rt, ru
        umull   \rt, \rd, \rx, \ry
        mul     \ru, \rt, \rm
        umlal   \rt, \rd, \ru, \rp
.endm
~~~

Note that the output of `MONTYMUL_1` is not fully reduced; it may
still be larger than `p`, though it is lower than `2*p`. To reduce
it further, into `[0, p-1]`, a conditional subtraction can be used:

~~~
@ rp = modulus p (odd integer, lower than 2^31)
@ rx < 2*p
@ rd <- rx mod p  (into range [0, p-1])
@ rp is preserved
@ rt is scratch
@ VARIANT: rd can be the same register as rx
.macro FINISHRED_1  rd, rx, rp, rt
        subs    \rd, \rx, \rp
        and     \rt, \rp, \rd, asr #31
        adds    \rd, \rt
.endm
~~~

Modular addition and subtraction can be done with `ADDMOD_2` and
`SUBMOD_2` (they were previsouly defined for a small modulus `p`,
but they can work with a 31-bit modulus, and if the inputs are
already in `[0, p-1]` then their output is in `[0, p-1]`).
