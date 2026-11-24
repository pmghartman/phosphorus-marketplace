# AArch64 code generation

Design notes for issue #9: a native arm64 backend next to the x86-64 one,
for Linux first (ELF), then macOS on Apple Silicon (Mach-O) and later
Windows on Arm. Linux and macOS are done — the encoder, instruction
selection (C structs by value included), register allocation, ELF and
Mach-O output; the Windows part waits for hardware to test on, and this
document records the decisions for it.

```
bic --arch arm64 build prog.bi     # an AArch64 Linux PIE
bic --target macos --arch arm64 build prog.bi   # an Apple Silicon Mach-O
bic run prog.bi                    # on arm64 Linux and Apple Silicon: native
```

## Status

| | |
|---|---|
| Encoder (`compiler/marm64`) | integer, load/store (all addressing forms), pairs, branches, ADR/ADRP, exclusives, acquire/release, LSE, scalar FP, the NEON the vector code uses (`neon.bi`); 2565 encodings checked against `llvm-mc` |
| Linux ELF | done: `elf.Program.Arch = elf.ARM64` |
| Mach-O (Apple Silicon) | done: the same program through `compiler/mmacho` (`macho_arm64.bi`), with Apple's C calling convention (`arm64_call.bi`); see Executable formats |
| Instruction selection | the whole IR, vectors in NEON (`arm64_simd.bi`, see Floating point and vectors): integer and float arithmetic, compares fused into branches (`cbz`/`cbnz` against zero) and `csel`, conversions, loads/stores with folded offsets and scaled indexes, shifted-register operands and `madd`/`msub`, division by constants without `sdiv`/`udiv`, inline small `memcpy`/`memset`, calls (AAPCS64, variadic, stack arguments), C structs by value in both directions (`arm64_call.bi`), atomics, switches with jump tables, stack slots aligned above 16 bytes |
| Register allocation | the shared iterated register coalescing allocator |
| Tested | every program of `tools/ci/check-programs.sh` at -O0..-O3, the backend test suite (`BIC_ARCH=arm64`) including 3000 random programs against the interpreter, C struct calls both ways against a gcc-built library (`TestCStructABI`, `TestCStructABIAAPCS`, `TestCFnPtrStructs`; so far under qemu-aarch64 only); on mulab-pi (Raspberry Pi 4, Cortex-A72, ARMv8.0, Debian 13), on mulab-pi5 (Raspberry Pi 5, Cortex-A76, ARMv8.2) and under qemu-aarch64. CI's `arm64` job (run by hand) runs on mulab-pi5: the linux-arm64 build of bic compiles and runs every program natively, and the encoder, ELF and backend tests and `tests/` run there with the backend suite compiled for arm64 |
| Not yet | Windows on Arm, which needs hardware to test on; it returns `backend.UnsupportedError` |
| Performance | see below; magic-number division, shifted-operand and `madd` folding and `cbz`/`cbnz` are done; no tail duplication, `ldp`/`stp` for adjacent accesses or LSE atomics yet |

`bench/` on mulab-pi (Raspberry Pi 4, Cortex-A72 at 1.8 GHz), best of
three:

| benchmark | bic -O2 | bic -O3 --unchecked | gcc -O2 | gcc -O2 -ffp-contract=off |
|---|---|---|---|---|
| fib(35), recursive | 0.080 s | 0.070 s | 0.076 s | 0.076 s |
| sieve, 50M | 3.72 s | 3.63 s | 3.34 s | 3.31 s |
| n-body, `vec3` math | 1.63 s | 1.50 s | 1.06 s | 1.11 s |

All three outputs are byte-for-byte those of gcc without floating-point
contraction; gcc's default on arm64 fuses `a*b+c` into `fmadd`, which
changes n-body's last digits. bic never contracts (the IR has no
fused multiply-add, and contraction would change results between targets).
n-body is where the missing folding shows most. (The table predates the
division and operand folding work, which removed about 1% of the
instructions in these programs; it hasn't been re-measured on the Pi.)

## Splitting the backend

What `compiler/mbe` already shares, unchanged in behaviour:

- **The MIR skeleton** (`mir.bi`): blocks, terminators (`tJmp`, `tBr`,
  `tSwitch`, `tRet`), operands (registers, immediates, memory with frame
  areas, symbols and the GOT), copies, calls with implicit uses and
  clobbers, and the 8-byte spill moves.
- **Instruction selection's skeleton** (`isel.bi`): block layout order,
  use counts and users, which values are absorbed into their uses (constants,
  addresses, compares that feed only a branch or select), phi elimination
  with parallel copies and critical-edge splitting (`phiCopies`).
- **Register allocation** (`regalloc.bi`): liveness, the interference graph,
  George/Appel coalescing, spill costs and spill rewriting. It sees an
  instruction only through `visitRegs` (which registers it uses and defines)
  and two predicates (is it a copy, is it a call), so nothing in it is
  x86-specific except the register file.
- **Frame slots** (`newSlot`, `fSlot`/`fInArg`/`fOutArg` operands): the
  emitters lay them out.
- **Data layout and linking** (`backend.bi`, now `modState.link`).

What is per target:

| | x86-64 | AArch64 |
|---|---|---|
| register file | 0..15 GPRs, 16..31 XMM | 0..30 X0–X30, 31 SP, 32..63 V0–V31, 64 XZR |
| opcodes | two-address `m*` forms with memory operands | three-address `ma*` forms (`arm64_mir.bi`) |
| selection | `isel*.bi`, `win64.bi` | `arm64_isel.bi` |
| allocation order, clobbers | SysV / Win64 lists | `a64Colors`, `a64CallClobbers` |
| emitter | `emit.bi` with `compiler/mx86` | `arm64_emit.bi` with `compiler/marm64` |

The first slice keeps one `mfunc`/`inst` type and gives each target its own
opcode range, so the allocator and phi elimination are reused as they are:
physical registers now occupy 0..71 (`firstVirt = 72`) and `mfunc.arm64`
selects the register file. `fnBuilder` dispatches its target-specific
entry points (`reg`, `constInto`, `cond`, `value`, `lowerParams`,
`lowerReturn`) on `modState.arm64`. Condition codes in the shared `condExpr`
hold `arm64.Cond` values on AArch64; both encodings invert by flipping the
low bit, which is all the shared code does with them.

The x86-64 output is byte-for-byte what it was (checked over every sample,
test and benchmark program for Linux, Windows and macOS at -O0, -O2, -O3).

Next step for the split, once a third target or more arm64 work makes it
pay: move the shared parts into an interface

```go
type target interface {
	physClass(r vreg) regClass       // register file
	colors(fp bool) [2][]int32       // allocation order
	callClobbers() []vreg
	selectValue(fb *fnBuilder, v *ir.Value) error
	selectCall / params / return / cond ...
	emit(mf *mfunc, used physRegs) error
}
```

with the per-target `inst` payload behind it, and code relocations as one
neutral `link.Reloc` type instead of `x86.Reloc` and `arm64.Reloc` side by
side in `elf.Program`.

## Encoder (`compiler/marm64`)

Pure functions return one 32-bit instruction word (`AddImm`, `LdStImm`,
`Fcsel`, ...); `Asm` collects them with labels (B/BL, B.cond, CBZ, ADR
fixups resolved in `Finish`), jump-table label differences and symbol
relocations (`RelAdrPage`, `RelAddLo12`, `RelLdstLo12`, `RelGotPage`,
`RelGotLo12`, `RelCall26`), which the linker applies with `arm64.Apply`.
SP and XZR are distinct `Reg` values even though both encode as 31, so each
encoder can reject the one its operand position doesn't take. Logical
immediates are encoded with LLVM's algorithm and tested exhaustively against
the Arm ARM's `DecodeBitMasks` (all 5334 64-bit and 1302 32-bit values).

`compiler/tests/arm64_asm.bi` assembles every case with `llvm-mc
-triple=aarch64 -mattr=+lse -show-encoding` and compares words; without
llvm-mc it compares with `compiler/tests/testdata/llvm-mc.golden`
(`BIC_UPDATE_GOLDEN=1 bic run --no-lint unittest.bi -run TestEncodings`
in `compiler/` rewrites it).

## Calling conventions

**AAPCS64 (Linux)**, used for Bismuth-internal calls too, as SysV is on
x86-64: integers and pointers in X0–X7, floats in V0–V7, further arguments
in 8-byte stack slots, results in X0 / V0, a large struct result's address
in X8 (not X0 as on SysV). Callee-saved: X19–X28, X29 (frame pointer), the
low 64 bits of V8–V15. Caller-saved: X0–X18, V0–V7, V16–V31. Variadic
arguments are passed like fixed ones; there is no AL register count.
Callees extend sub-word integer arguments themselves; bic extends them at C
call sites anyway (Apple requires it, see below), and its own functions
never rely on the upper bits.

X16/X17 are the emitter's scratch registers (the linker may also use them
in veneers), X18 is never allocated (reserved on Apple and Windows), X29
always holds a frame record when a function has a frame, so debuggers and
`perf` unwind it.

**Apple arm64** differs from AAPCS64 in ways the macOS port follows:

- Variadic arguments always go on the stack, in 8-byte slots, whatever
  their type; only the fixed arguments use registers. `printf` and
  `snprintf` in std depend on this.
- Stack arguments are packed at their natural size and alignment (an `i8`
  takes one byte), not one 8-byte slot each.
- The caller sign- or zero-extends `i8`/`i16` arguments (and `bool`) to 32
  bits; the callee extends return values.
- X18 is reserved; X29 must always be a valid frame pointer.
- `char` is signed; `long double` is `double`.

`a64Classify` takes them into account for C calls and C-callable
functions on macOS (`Counters.apple`): arguments from the IR's fixed
parameter count on (`Sig.params`, when `Sig.variadic`) go to 8-byte stack
slots, and other stack arguments take their natural size and alignment
(`ArgLoc.stSize`). Calls between Bismuth functions keep the Linux layout,
as both sides are bic's. A C struct passed as a variadic argument is
unsupported.

So a C function that is variadic in C must be declared variadic in
Bismuth too (`@cname("open") extern c fn c_open(*u8 path, i32 flags,
...)`): declared with a fixed third parameter, `open`'s mode or
`fcntl`'s argument would go in a register on Apple Silicon while the
callee reads it from the stack. It works either way on Linux and x86-64,
so nothing else catches it.

**C structs by value** (done, `arm64_call.bi`). The frontend describes
each struct argument and result target-neutrally in `ir.ArgInfo` and
`ir.Sig`: its size, its alignment and, when it is a homogeneous
floating-point aggregate, an `ir.HFA` (one to four `f32` or `f64` members
of one type through nested structs and arrays, with their offsets). The
SysV eightbyte classes stay beside them for x86-64. AAPCS64 then:

- passes an HFA in consecutive V registers, one member each, whatever its
  size (a 32-byte `{double a, b, c, d}` takes D0–D3); when they don't all
  fit, the struct is copied to the stack and no later float argument uses
  a register;
- passes other structs of up to 16 bytes in one or two X registers (a
  16-byte aligned one from an even register), or on the stack once the
  registers run out, closing them to later arguments;
- copies larger structs in the caller and passes the copy's address;
- returns an HFA in V0–V3 (also one over 16 bytes, which the IR returns
  through a `RetMem` pointer: the backend stores the registers there),
  other structs of up to 16 bytes in X0/X1 and larger ones through the
  address in X8.

Both directions are covered: calls to `extern c fn`s and C function
pointers, and C calling into Bismuth functions with the C ABI (the
backend side; the frontend still accepts only scalar parameters and
results for `export c fn`). Last registers of odd-sized structs (7, 12
bytes) are loaded and stored piecewise, never past the struct. C unions
are passed as plain composites, not HFAs.

**Windows ARM64** (later) is AAPCS64 with X18 reserved for the TEB,
variadic floats passed in integer registers, and a PE/COFF image with
ARM64 base relocations; `compiler/mpe` would grow the same `Arch` switch as
`compiler/melf`.

## Addressing

Everything is PC-relative, so executables are PIE without text relocations:

- internal data and functions: `adrp xN, sym; add xN, xN, :lo12:sym`, and
  loads/stores fold the low 12 bits: `adrp x16, sym; ldr x0, [x16,
  :lo12:sym]` (the scaled form needs `sym+off` aligned to the access size,
  which the emitter checks with the global's alignment, else it adds first);
- imports: `adrp x16, :got:sym; ldr x16, [x16, :got_lo12:sym]`, then `blr
  x16` for calls. There is no PLT: like the x86 backend, everything is
  bound at load time (`BIND_NOW`, RELRO GOT);
- calls between Bismuth functions are `bl` to a label (±128 MB);
- jump tables are 32-bit offsets from the table, reached with `adr`.

ADRP reaches ±4 GB, the small code model, which covers any executable bic
writes.

## Executable formats

**ELF (done).** `elf.Program.Arch` selects `EM_AARCH64` (183), an AArch64
`_start` that calls `__libc_start_main` (the loader passes `rtld_fini` in
X0 and leaves `argc` at `[sp]`), `/lib/ld-linux-aarch64.so.1`, the
dynamic relocations `R_AARCH64_GLOB_DAT`, `R_AARCH64_RELATIVE` and
`R_AARCH64_ABS64`, and segments aligned to 64 KB: arm64 Linux kernels run
with 4, 16 (Apple M-series under Asahi, Raspberry Pi 5) or 64 KB pages, and
an executable aligned for 4 KB fails to load on the others.

**Mach-O (done).** `compiler/mmacho` has the same `Arch` switch
(`macho_arm64.bi`): `CPU_TYPE_ARM64` (0x0100000c), 16 KB pages and segment
alignment, the AArch64 code and relocations patched as for ELF (from the
image's addresses, which ADRP's pages need), and imports bound through
`__DATA_CONST,__got`.

- *Symbol names.* std names the 64-bit-inode calls as the SDK does on
  x86-64 (`stat$INODE64`, `readdir$INODE64`); arm64 libSystem has only the
  plain names, so the arm64 writer binds those (`dyldName`).
- *Starting fibers.* The scheduler enters a new fiber's stack with
  `makecontext` and `setcontext` elsewhere; Apple does not support them on
  Apple Silicon, so `std/core/sched_darwin_arm64.bi` calls
  `bic_fiber_enter(fn, sp)` instead, five instructions (`mov sp, x1`;
  clear x29 and x30; `blr x0`; `brk`) the writer appends to the code of a
  program that imports it. std files for one CPU end in `_arm64` or
  `_amd64`, after any OS (`name_darwin_arm64.bi`), and the checker picks
  them by the target's CPU.

- *Code signature.* Mandatory on Apple Silicon: the kernel kills an
  unsigned arm64 process. macho already writes an ad-hoc SHA-256 signature
  (`codesign.bi`); for arm64 the page size it hashes stays 4 KB
  (`CS_PAGE_SIZE`), the signature must cover the final file, and any
  post-link edit invalidates it.
- *Fixups: `LC_DYLD_INFO_ONLY` vs `LC_DYLD_CHAINED_FIXUPS`.* Keep the
  classic `LC_DYLD_INFO_ONLY` rebase/bind opcodes, which macho writes
  today for x86-64. dyld on every macOS version that runs on Apple Silicon
  (11+) still accepts them for arm64 executables, and they reuse the
  existing writer and its tests. Chained fixups (macOS 12+, the default of
  Apple's `ld` since then) are smaller and page in lazily, which matters
  for large binaries with many pointers, not for bic's; adopt them when
  dyld or the App Store starts requiring them, as a separate encoder for
  the same list of rebases and binds.
- `LC_BUILD_VERSION` with platform macOS and a minimum of 11.0;
  `LC_MAIN` as today (dyld calls `main`, so no start stub).

The release packages `bic-*-macos-arm64` as an arm64 bic built with
`--target macos --arch arm64`, which builds arm64 programs by default.
Testing needs the macOS runner (`macos-15`, host macbook): `ci`'s and
`release`'s macOS jobs, ticked when started, run every program natively
there with that bic.

## Atomics

The IR's atomics are sequentially consistent. The first slice uses the
ARMv8.0 forms, which every arm64 CPU has:

| IR | AArch64 |
|---|---|
| `AtomicLoad` | `ldar` |
| `AtomicStore` | `stlr` |
| `AtomicAdd`, `AtomicXchg` | `ldaxr` / `stlxr` loop |
| `AtomicCAS` | `ldaxr; cmp; b.ne; stlxr; cbnz` loop, `clrex` on failure, `cset eq` |
| `Fence` | `dmb ish` |

`ldar`/`stlr` are what C11 compilers emit for seq_cst loads and stores
(they are ordered against each other, unlike x86's plain moves). The loops
are correct everywhere but contend badly: under contention LL/SC retries,
and on large many-core machines LSE (ARMv8.1 `ldaddal`, `swpal`, `casal`,
already in the encoder) is several times faster. Apple M1 and later,
Graviton 2+, Ampere and Cortex-A76+ have LSE; the Raspberry Pi 4's
Cortex-A72 (mulab-pi) does not. Plan: on macOS always use LSE (every Apple
Silicon Mac has it); on Linux keep LL/SC by default, and select LSE with a
`--cpu` flag or at load time through an outlined helper that checks
`AT_HWCAP & HWCAP_ATOMICS` once, as libgcc's `-moutline-atomics` does.

## Floating point and vectors

Scalars live in V registers as S (f32) or D (f64). Float compares use
`fcmp` and the conditions that are false on unordered operands: `mi`
(less), `ls` (less or equal), `gt`, `ge`, `eq`; `ne` is true when
unordered, as the IR's `fne` wants; `vs`/`vc` test unordered/ordered.
`FMin`/`FMax` map to `fminnm`/`fmaxnm`, which return the other operand for
a quiet NaN, like C's `fmin`/`fmax`. Constants are `fmov` immediates when
they fit (±n/16·2^e), `fmov dN, xzr` for zero, else loads from the
constant pool.

Vectors are NEON (`compiler/mbe/arm64_simd.bi`, #426). An F32X4, F64X2
or I32X4 lives in a whole V register (q), an F32X2 in its low 64 bits
(d); an I32X8 (std's `u32x8`, `u8x32`, `i16x16`, `i32x8`) is a pair of
q registers: `mir/halve.bi` splits each I32X8 value into its two 128-bit
halves before selection, which works because every I32X8 operation acts
on each half alone (AVX2's lanes), and the half and join operations
only pick halves. Loads and stores are `ldr`/`str q` (`d`), copies `mov
v.16b`, spills 16-byte slots; the callee-saved V8–V15 keep only their
low 64 bits across a call, so a vector live across one interferes with
them too and is spilled (regalloc's `wideClobbers`). The vectorizers run
for arm64 as for x86-64, and `core.simdNative()` is true there, so
std's SIMD types and ChaCha20's vector paths are NEON code;
`u32x8.native()` and `u8x16.ssse3()` are true (std/core/simd_arm64.bi).

| IR | NEON |
|---|---|
| float `FAdd` .. `FDiv`, `FNeg`, `FAbs`, `FSqrt` | `fadd` .. `fdiv`, `fneg`, `fabs`, `fsqrt` (`.2s`, `.4s`, `.2d`) |
| float `FMin`, `FMax` | `fminnm`, `fmaxnm`, as the scalar code |
| `Add`, `Sub`, `And`, `Or`, `Xor` of `i32x4` | `add`, `sub .4s`; `and`, `orr`, `eor .16b` |
| shifts of `i32x4` (and the 16-bit `VOp`s) | `shl`, `ushr`, `sshr #n`; by a register `ushl`/`sshl` by the count dup'ed (negated for a right shift) |
| a rotate (`u32x4.rotl`) | `shl` + `sri`; by 16 `rev32 .8h` |
| `VShuffle` | `zip1`, `zip2` of `.4s` or `.2d` |
| `Select` | `csetm`, `dup .2d`, `bsl` |
| `VBuild`, `VSplat` | `dup` and `ins`; constants `movi` or a constant pool `ldr q` |
| `VExtract` | `umov` (i32), `dup` to a scalar register (floats) |
| `add8/16`, `sub8/16`, `mul16/32` | `add`, `sub`, `mul` |
| `addsatu8`, `subsatu8`, `addsats16`, `subsats16` | `uqadd`, `uqsub`, `sqadd`, `sqsub` |
| `mulhis16` | `smull`, `smull2`, `uzp2` (the products' high halves) |
| `madd16` | `smull`, `smull2`, `addp .4s` |
| `sadu8` | `uabd`, then `uaddlp` to 16, 32 and 64 bits (psadbw's two sums) |
| min, max | `umin`/`umax .16b`, `smin`/`smax .8h`, `.4s` both |
| compares | `cmeq`, `cmhi` (unsigned), `cmgt` (signed) |
| `packs32`, `packu16` | `sqxtn` + `sqxtn2`, `sqxtun` + `sqxtun2` |
| zips | `zip1`, `zip2`; with zeros (`widenLo`/`widenHi` of `u8x16`) `uxtl`, `uxtl2`; `sra(zipLo16(a, a), 16)` (`i16x8.widenLo`) `sxtl` |
| `abs16`, `abs32` | `abs` |
| `signs16`, `signs32` | `cmlt #0`, `cmeq #0`, `eor`, `sub`, `bic` |
| `hadd16`, `hadd32` | `addp` (phadd's order) |
| `shuf8` | `tbl` of `idx & 0x8f`: pshufb uses `idx & 15` unless bit 7 is set, tbl gives 0 past 15 |
| `shrbytes`, `shlbytes` | `ext` with a zero register |
| `from64`, `to64` | `ldr d` (a load's) or `fmov d, x`; `str d` (a store's) or `fmov x, d` |
| `add64`, `sub64` (`u64x2`) | `add`, `sub .2d` |
| `shl64`, `shr64` | `shl`, `ushr .2d #n`; by a register `ushl .2d` as above |
| `mulu32` (`u64x2.mulLo32`) | `xtn .2s` of each operand, `umull .2d` |
| `alignr` | `ext` of the pair |
| `clmullo`, `clmulhi` | `pmull`, `pmull2` (`.1q` of `.1d`, `.2d`) |
| `aese`, `aesmc` | `aese`, `aesmc` (x86-64's `aesenc` and `aesenclast` are scalarized) |
| `sha256h`, `sha256h2`, `sha256su0`, `sha256su1` | the SHA2 instructions (x86-64's SHA VOps are scalarized) |

A function with a vector value the selection doesn't cover
(`arm64_simd.lowerable`) is scalarized instead (`mir/scalarize.bi`), as
everything was before; nothing reaches that today. The vectorizers
decide which `vec2`/`vec3`/`vec4` code becomes vectors, as on x86-64.

## Next

1. Windows on Arm (PE/COFF ARM64), waiting for hardware.
2. LSE atomics on macOS, where every CPU has them (see Atomics).
3. Performance on `bench/`: `tbz`/`tbnz` for single-bit tests (with a
   fallback when the 32 KB range doesn't reach), `ldp`/`stp` for adjacent
   accesses and memcpy, tail duplication of loop tests as on x86, and LSE
   atomics behind an `AT_HWCAP` check (see Atomics). Done: magic-number
   division by constants, `madd`/`msub` and shifted-operand folding
   (`add x0, x1, x2, lsl #3`), `cbz`/`cbnz` for compares with zero; `stp`/
   `ldp` already save and restore the frame record and callee-saved pairs.
4. The target interface described under "Splitting the backend".
