# sha3

SHA-3 and SHAKE for [sysl](https://github.com/sysl-lang/sysl) — the four FIPS 202 digests and both
extendable-output functions, bound to [tiny_sha3](https://github.com/mjosaarinen/tiny_sha3).

```sysl
import sh.sysl.sha3.{sha3_256, shake256}
import sysl.encoding.hex_string

val digest = sha3_256("abc".bytes)

print(hex_string(digest))

var key: []u8 = [0; 64]

shake256(secret, key)
```

**Nothing has to be installed to use this.** tiny_sha3 is one C file that includes `<stddef.h>` and
`<stdint.h>` and nothing else, and this package carries it: sysl compiles a library's C as part of the
build, so there is no `-l` flag, no `pkg-config`, and no build script anywhere in this repository.

## It runs where nothing else here does

Every vendored-C package in this organisation declares `requires {}` when it neither allocates nor
asks the operating system for anything. This one makes a second claim that the others cannot:

```
sysl build-lib . --target thumb-freestanding
```

succeeds. qrcodegen, qoi and miniz all make the same capability claim and all fail that command,
because clang gives a freestanding target `stdint.h` and `stddef.h` and **not** `assert.h`, `stdlib.h`
or `string.h`. tiny_sha3 reaches for no C library function at all — it copies with explicit loops —
so the two headers it wants are the two it gets.

So this is usable on a microcontroller with no libc, which for a hash function is the case that comes
up: verifying a firmware image before jumping to it.

## Nothing here asks you for storage

A sponge's state is 216 bytes, and every other caller-placed binding in this organisation would have
made that your problem — miniz's `Deflater` is *placed* in a slice you own, because sysl has no
allocator to reach for.

**This one does not have to, because SHA-3's state holds no pointers.** It is 25 lanes of 64 bits and
three counters, so it can be copied from one address to another and keep working. That lets the state
live inside a `Hasher` as a field, which makes a `Hasher` an ordinary sysl value: it goes in a `var`,
it can be passed to a function and returned from one, and there is no storage argument, no alignment
to get right, and nothing that has to outlive anything.

The `hasher survives being moved` test is what pins that reading, and it is why it exists.

The state is declared as an array of `u64` rather than of bytes, and that is alignment's whole answer:
`sha3_ctx_t` reads its state as `uint64_t q[25]`, so it wants an 8-aligned address, and an array of
`u64` has one by construction where a `[216]u8` would promise nothing. The raw layer asserts
`_Alignof(sha3_ctx_t) == 8` so that a change on the C side fails a test rather than corrupting a
digest.

## What you get

| | |
|---|---|
| `sha3_224(data)` `sha3_256(data)` `sha3_384(data)` `sha3_512(data)` | one call, answers the digest by value |
| `shake128(data, out)` `shake256(data, out)` | fill `out`, however long it is |
| `hasher(size)` → `Hasher.push` / `.finish` / `.reset` | a message that arrives in pieces |
| `shake(rate)` → `Shake.push` / `.squeeze` | absorb, then squeeze as much as you want |

`Size` is `Bits224`, `Bits256`, `Bits384` or `Bits512`; `Rate` is `Shake128` or `Shake256`. Both
constructors default to the 256-bit form, so `hasher()` is SHA3-256 and `shake()` is SHAKE256.

**The four digest lengths are four algorithms, not truncations of one.** The length sets the sponge's
rate to `200 - 2 × len`, so SHA3-256 and the first 32 bytes of SHA3-512 disagree from the very first
byte. There is a test that says so.

**A SHAKE has no natural output length**, which is why `shake256` takes the buffer instead of
answering one: ask for 16 bytes or 1,000 and both are equally correct. Squeezing twice continues the
stream rather than restarting it, so 32 bytes then 32 more gives exactly what asking for 64 gives.

## Nothing can fail

There is no `Result` anywhere in this package. SHA-3 consumes any input and always produces exactly
its digest length, so the only ways to misuse it are contract violations — finishing a hasher twice,
pushing to a shake that has already been squeezed, handing `finish` a buffer of the wrong length —
and those `require` and stop rather than returning something a caller might ignore.

## The tests are somebody else's numbers

Every expected value in `tests.sysl` comes from **two independent implementations that agree**,
Python's `hashlib` and OpenSSL's `dgst`, and the short ones are the values FIPS 202 itself prints.
Nothing is checked against a value this binding produced.

Two oracles rather than one is deliberate: a lone disagreement between a binding and a single
reference is indistinguishable from a bug in the binding, and an hour goes into the wrong half.

The vectors cover the empty message, `abc`, and 1,000 bytes of `a` — the last because it is longer
than all four rates and so forces the sponge to permute repeatedly, which is where a dropped length or
an unadvanced pointer shows up. The behavioural tests cover what no vector can: that splitting a
message does not change its digest, that a squeezed stream continues, and that a `Hasher` survives
being copied.

```
30 passed, 0 failed
```

## Layout

```
sh/sysl/sha3/
    sha3.sysl           the binding
    tests.sysl          the published vectors and the behavioural tests
    c/
        c.sysl          tiny_sha3 as C declares it
        sha3.c          vendored from mjosaarinen/tiny_sha3 at dcbb319, unmodified
        sha3.h
        tests.sysl      the ABI facts the layer above depends on
package.hocon
```

**Everything that is C lives in `c/`**, module `sh.sysl.sha3.c`. That is the two-layer shape every
binding in this organisation uses: the `c` layer has to be *faithful*, and the layer above it has to be
*pleasant*, which is a different question and would otherwise be answered in the same breath.

**There is no `shim.c`, which is unusual here.** Upstream's three macros — `shake128_init`,
`shake256_init` and `shake_update` — all expand to a call the `c` layer already declares, so the
pleasant layer simply makes it. A macro needs a shim when it has no symbol to link *and* no spelling
in terms of one; these have the second.

The vendored C is unmodified. `KECCAKF_ROUNDS` stays at 24 because FIPS 202 says Keccak-f[1600] has
24 rounds — it is a knob for people studying reduced-round Keccak, and moving it would stop this
being SHA-3.

## Using it

Name it in your project's `package.hocon` and `sysl build` fetches it:

```hocon
dependencies {
  sha3 { git = "github.com/sysl-lang/sha3", version = "0.1.0" }
}
```

## Licence

tiny_sha3 is MIT, © 2015 Markku-Juhani O. Saarinen; `LICENSE` is upstream's, carried unchanged. The
binding is offered under the same terms.
