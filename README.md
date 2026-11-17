# gmp

Integers and rationals of any size, for [sysl](https://github.com/sysl-lang/sysl), bound to
[GMP](https://gmplib.org/), the GNU Multiple Precision Arithmetic Library.

*An independent sysl package for GMP. It is not affiliated with or endorsed by GMP's authors or trademark holders.*

```
dependencies {
  gmp { git = "github.com/sysl-lang/gmp", version = "0.1.0" }
}
```

```
import sh.sysl.gmp.*

val n = Int.factorial(100)
val p = Int.power(2, 127) - 1

print(n / Int.factorial(98))                         // 9900
print(p.is_probable_prime())                         // ProbablyPrime
print(Int.of(3).pow_mod(p - 1, p).unwrap())          // 1
print(Int.parse("ff", 16).unwrap() + 1)              // 256
print(Rat.of(1, 3) + Rat.of(1, 6))                   // 1/2
```

`Int` is an integer with no size limit and every operator a built-in integer has (`+ - * / %`,
`- ~`, `& | ^`, `<< >>`, the comparisons), with a `long` allowed on either side of the arithmetic
ones. `/` and `%` truncate toward zero as a built-in integer's do; the floor, ceiling and Euclidean
divisions are methods. `Rat` is an exact rational kept in lowest terms. `Random` draws either from a
seeded generator.

## Install GMP

```
brew install gmp              # macOS
sudo apt install libgmp-dev   # Debian / Ubuntu
```

The package finds it through `pkg-config` (`gmp`). It is not vendored: GMP's speed is the assembly
its own `configure` picks for the processor, which a copy compiled as plain C would give up, and the
copy a CVE watch cares about is the system's.

Tested against **GMP 6.3.0**.

## An `Int` is a handle

The digits live in one counted box, freed when the last `Int` naming it goes, so an `Int` is copied,
stored and passed like a number. **Every operator and every ordinary method builds a fresh `Int`**
and leaves its operands alone, so code written with operators never sees a value change under
another name.

For hot loops there is an in-place family, shaped like GMP's C API: `x.add_assign(y)`,
`x.mul_assign(k)`, `x.add_mul(a, b)` (x += a·b), `x.sub_mul`, `x.div_assign`, `r.set_mul(a, b)`,
`r.set_add`, `r.set_div`, `x.assign(v)`, `x.set_bit(i)` and the rest. These change the value **every
copy of that `Int` sees**, as a write to a `Buf` does; `copy()` makes an independent one first.
`Int.with_capacity(bits)` allocates up front. The pidigits spigot in `tests_pidigits.sysl` is written
both ways.

## Errors

- **Dividing by zero stops the program** with `Int division by zero` (or `Rat division by zero`), as
  `/` on a built-in integer traps — GMP itself would raise `SIGFPE`. `checked_div` and `checked_rem`
  answer `None` instead, and `Rat.from` and `Rat.inv` answer `None` for a zero denominator.
- **`parse` answers a `ParseError`**: `InvalidBase` (not 0 or 2 to 62), `Empty`, `InvalidDigits` (a
  stray character, a misplaced sign, an embedded NUL), and for a `Rat`, `ZeroDenominator`.
- **Where GMP's answer is undefined or does not exist, the method answers `None`**: `sqrt` and
  `root` of a negative (an even root), `invert` with no inverse, `pow_mod` with a zero modulus or a
  negative exponent and no inverse, `pow_mod_secure` with an even modulus, `jacobi` with an even or
  non-positive `b`, `prev_prime` below 3, `remove_factor` by 0 or ±1, `count_ones` of a negative, and
  `to_long`/`to_ulong`/`to_int` of a value that does not fit.
- An invalid base for `to_string_base` stops the program, naming the base.

## What is bound

`sh.sysl.gmp.c` binds **the whole `mpz` integer layer** (every function `gmp.h` names, 151
externs), **the whole `mpq` rational layer** (32), the random-state functions (10), the allocator
hooks, and the three exported variables (`gmp_version`, `mp_bits_per_limb`, `gmp_errno`). Every
public name in `gmp.h` is a macro over a `__gmpz_*` / `__gmpq_*` / `__gmp_*` symbol, so each `extern`
names the real export. `Mpz`, `Mpq` and `RandState` are laid out field for field and checked against
the header's `sizeof` and `offsetof`. There is no C in the package: the function-like macros
(`mpz_sgn`, `mpz_odd_p`, `mpz_even_p`, `mpq_sgn`, `mpq_numref`, `mpq_denref`) only read those fields,
and the layer above reads them the same way.

`sh.sysl.gmp` is what an application names:

- **`Int`**: `of`, `parse` (any base 2–62, or 0 for C prefixes), `from_real`, `from_bytes_be`/`_le`,
  `with_capacity`; `factorial`, `double_factorial`, `multifactorial`, `primorial`, `binomial`,
  `power`, `fibonacci`/`fibonacci2`, `lucas`/`lucas2`; `to_long`, `to_ulong`, `to_int`,
  `low_ulong`, `to_real`, `to_real_exp`, `to_string_base`, `to_bytes_be`/`_le`, `bit_length`,
  `size_in_base`, `limb_count`, `limb`, `sign`, `is_zero`, `is_negative`, `is_odd`, `is_even`;
  `compare` and `compare_abs` against `Int`, `long`/`ulong` and `real`; `abs`, `pow`, `pow_mod`,
  `pow_mod_secure`, `sqrt`, `sqrt_rem`, `root`, `root_rem`, `is_perfect_square`,
  `is_perfect_power`; `div_floor`, `div_ceil`, `rem_floor`, `rem_ceil`, `modulo`, `div_exact`,
  `div_rem`, `div_rem_floor`, `div_rem_ceil`, `checked_div`, `checked_rem`, `mod_ulong`,
  `shr_trunc`, `shr_ceil`, `rem_pow2` (and `_trunc`, `_ceil`), `is_divisible_by`,
  `is_divisible_by_pow2`, `is_congruent`, `is_congruent_pow2`; `gcd`, `gcd_ext`, `lcm`, `invert`,
  `jacobi`, `legendre`, `kronecker`, `is_probable_prime`, `next_prime`, `prev_prime`,
  `remove_factor`, `choose`; `bit`, `with_bit_set`/`_cleared`/`_flipped`, `count_ones`,
  `hamming_distance`, `find_zero`, `find_one`, `trailing_zeros`; the in-place family above; `copy`;
  `as_mpz` for C. `Eq`, `Ord`, `Hash`, `Display` (honouring width) and `Debug`.
- **`Rat`**: `of`, `from`, `from_real`, `parse`, `num`, `den`, `sign`, `is_zero`, `is_integer`,
  `to_real`, `to_string_base`, `abs`, `inv`, `mul_pow2`, `div_pow2`, `floor`, `ceil`, `trunc`,
  `compare`, `copy`, `as_mpq`; `+ - * /` with a `Rat` or a `long`, unary `-`; `Eq`, `Ord`, `Hash`,
  `Display`, `Debug`.
- **`Random`** (a `&Random`): `new`, `mersenne`, `congruential`, `congruential_of_size`, `copy`,
  `reseed`, `bits`, `below`, `sparse_bits`, `word_bits`, `word_below`, `as_randstate`. Not for
  cryptography.
- `version()` and `limb_bits()`.

**Deliberately absent**, each for its reason:

- **the `mpf` floating-point layer** (68 functions) — MPFR supersedes it and GMP's manual recommends
  MPFR for new code; that is a binding of its own;
- **the `mpn` limb-array layer** — the low-level kernels `mpz` is built on, which work on arrays a
  caller sizes by hand; `mpz` exists to hide exactly that (`mpz_limbs_*` and `mpz_roinit_n` are
  bound, for a caller who needs the limbs);
- **the `gmp_printf` / `gmp_scanf` families** — variadic formatting over C's `printf` grammar;
  `Display`, `to_string_base` and `parse` already render and read every value;
- **the variadic `mpz_inits` / `mpz_clears` / `mpq_inits` / `mpq_clears`** — conveniences over
  `init` and `clear`, which are bound;
- **what the manual calls obsolete or internal**: `mpz_random`, `mpz_random2`, `mpz_array_init`,
  `_mpz_realloc`, `gmp_randinit`, `mpz_millerrabin`, and the `__gmpz_*` helpers no public macro names;
- **`mpz_set_f` and `mpq_set_f`**, which take an `mpf`.

The `FILE *` functions (`mpz_out_str`, `mpz_inp_str`, `mpz_out_raw`, `mpz_inp_raw`, `mpq_out_str`,
`mpq_inp_str`) are bound in `sh.sysl.gmp.c` and tested against a `tmpfile()`, but have no pleasant
form: a sysl program renders and parses strings, and serializes with `to_bytes_be`.

## Tests

`sysl test .` runs 94 tests. Oracles that do not come from GMP: the first 1,000 digits of π (the
Benchmarks Game's pidigits spigot, in-place and with operators, against digits computed separately by
Machin's formula over Python's integers), 100!, F(100), C(100, 50), RSA-100 and its published factors,
textbook RSA (p = 61, q = 53, e = 17, d = 2753), the Mersenne primes 2¹²⁷ − 1 and 2⁸⁹ − 1 and the
factorization of 2⁶⁷ − 1, H(10) = 7381/2520, Jacobi symbols from the literature, and the four
operations, the bit operations and the shifts cross-checked against sysl's own `i512` and `u256`. The
structs are checked against the header's layout. Every error path above has a test, including each
division by zero (`should_trap`).
