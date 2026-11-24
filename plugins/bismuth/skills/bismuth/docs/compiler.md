# bic internals

bic is written in Bismuth and builds itself; its sources are in
`compiler/` ([compiler/README.md](../compiler/README.md) has the module
layout, how to build it and how to run its tests).

```
.bi source ──lexer/parser──▶ AST ──checker──▶ typed AST ──lowering──▶ Bismuth IR
                                                                         │
    ELF / PE / Mach-O executable ◀──x86-64 / arm64 backend◀──optimizer◀───┘
```

## Frontend (`compiler/lexer.bi`, `parser.bi`, `checker.bi` and its `c*.bi` parts)

- **Lexer/parser**: newline-insensitive, `;`-terminated grammar from the spec.
  String interpolation is parsed at lex time (`"{expr:spec}"`). Generic `<`
  is disambiguated by speculative parsing (`List<int>()`, `a < b`), and
  `>>` is two adjacent `>` tokens, so `List<List<T>>` needs no spaces.
- **Checker**: bidirectional local inference (literals adapt to the expected
  type; there are no implicit conversions otherwise), monomorphized
  generics (each instantiation is checked separately after cloning the
  generic body), overloaded methods (`vec3.new(f32)` / `vec3.new(f32, f32,
  f32)`), operator methods (`opAdd`, `opMul`, `opRMul`, `opNeg`, `opCmp`),
  smart-pointer `deref`, flow-sensitive null narrowing of locals and local field paths (`if x != null`,
  `if x == null { return; }`), match exhaustiveness, `ref` checks at call
  sites, the `parallel for` independence rule, and the iteration protocol
  (`len` + `slot` for indexable collections, `next` for iterators).
- **C headers** (`cimport.bi`, `bindgen.bi`): the header is preprocessed
  with `cc -E` and the target's OS macros (`cTargetFlags`), and the
  declarations (functions, structs, unions, enums, typedefs) are parsed
  with the file each came from, from the line markers. C types are
  mapped with the target's sizes (`long` is 4 bytes on Windows).
  `bic bindgen` writes the declarations from the header's own directory as
  Bismuth source (`Bindgen`). `import c` uses them directly, and evaluates
  macros it can't read (`#define RED (Color){...}`) by compiling a small
  probe once at import time. Without a C compiler, `import c` reports how to
  generate bindings instead. See [c-interop.md](c-interop.md).

## Lowering (`compiler/lower.bi` and `lexpr`, `lstmt`, `lcall`, `lvirt`, `lserde`, `ldebug`)

Every Bismuth type is either a scalar (an SSA value) or an aggregate that
lives in memory and is handled by address. Each expression result is either
*owned* (a fresh value the consumer moves or drops) or *borrowed* (an
address of an existing place). Copies happen only when a borrowed value
must become owned, through generated copy glue (string refcount
increments, user `clone` hooks). Drops and `defer`s run at scope exit on
every path, including early `return`, `break`, `continue` and `?`. Early
exits from the middle of an expression jump through per-statement cleanup
stubs. Returning a local moves it.

Generated per-type glue: `drop`, `copy`, `eq`, `hash`, `fmt` (interpolation),
`bytes`/`frombytes` (packed structs with byte order) and `enc`/`dec` (the
binary wire format).

### Inheritance and virtual methods (`cvirt.bi`, `lvirt.bi`)

A struct that extends another starts with the base's whole layout (its
fields are the base's `Field`s, at the same offsets), so passing it by
`view`/`ref` as the base, or calling an inherited method, passes the same
address; the checker records the upcast as a contextual conversion, which
lowering applies without a copy. Before any function is checked,
`resolveFamilies` resolves every user struct, so each knows every struct
that extends it, and `resolveVirtuals` builds each struct's slots: the
root's virtual methods first, each with the method the struct uses (its
own, inherited, or a required one with no body).

A struct with virtual methods has a vtable pointer at offset 0. Vtables
are read-only globals with relocations (RELRO in ELF): the base's vtable,
the exact drop glue (`dropx`) and one function per slot. Owned values
always hold their exact type's vtable: struct literals and wire decoding
store it, and `copyInto` stores the destination type's, so a copy made
from an extending struct is a value of the base and nothing else. Only
places reached by address can hold an extending struct, and only those
dispatch:

- **Calls.** A call of a virtual method is direct when every struct that
  can be behind the receiver (its static type and everything extending
  it, the ones that can be built) uses the same method; the program is
  compiled whole, so this is the common case. Otherwise, when at most six
  structs can be behind it, the call compares the receiver's vtable
  pointer with each one's (all but the last, the only one left) and calls
  that struct's method directly. Beyond six it loads the slot from the
  vtable and calls it indirectly; a struct literal stores the vtable's
  address, so after inlining, store-to-load forwarding and
  `loadConstGlobal` turn that load into a `FuncAddr` and the call into a
  direct, inlinable one.
- **Drops.** The drop glue of a struct with virtual methods that something
  extends calls the vtable's `dropx` (none for a zeroed, moved-out value),
  which runs each level's `drop` hook from the most derived and drops
  every field. `Shared<Derived>` converts to `Shared<Base>` with no change
  of bits, and the box's value is dropped through it this way.
- **Live values.** Moving a value out zeroes its source, so a struct with
  a `drop` or `clone` hook carries a hidden trailing live byte that making
  the value (a struct literal, decoding, serde) sets to 1; the hooks run
  only when it is set, whatever the value's other bytes. A struct with a
  vtable uses the vtable pointer instead, and one marked `@zeroIsEmpty`
  (`List`, `Map`, `StrBuf`, `Shared` and the other `std/core` handles, whose
  all-zero value is empty or never live) or packed has no live byte and
  skips the hooks for an all-zero value.
- **Assignment** into a struct with virtual methods drops and replaces
  only the static type's part and keeps the vtable pointer, so writing a
  `Component` through a `ref Component` that holds a `Spinner` leaves a
  `Spinner`.
- **`is`/`as`** compare the vtable with the target's, walking up the
  bases' vtables when the target has extensions of its own.

Bounds checks are emitted in one canonical shape (`ICmp uge i, n` → a
no-return `panicIndex` block) so the optimizer can prove them away.

## IR (`compiler/mir`)

SSA over `i1 i8 i16 i32 i64 f32 f64 ptr` and the SIMD vector types
`f32x2 f32x4 f64x2 i32x4 i32x8`, with explicit memory (`Alloca`, `Load`, `Store`,
`PtrAdd`, `Memcpy`), calls with SysV C-ABI annotations for struct-by-value
arguments and returns, and atomics. The float vectors are made by the
vectorizers: the float arithmetic ops (`FAdd` .. `FMax`, `FNeg`, `FSqrt`,
`FAbs`) work lane by lane on them. `i32x4` and `i32x8` come from the
`core.simd*` intrinsics behind std.core's `u32x4` and `u32x8` (ChaCha20
uses them): `Add`, `Sub`, `And`, `Or`, `Xor` work lane by lane, the
shifts move every lane by a scalar amount (mod 32), and `VShuffle [mask]`
picks lanes of two vectors (the 4x4 transpose's four interleaves, which
an `i32x8` applies to each 128-bit half, and for `i32x8` the joins of
halves). `VOp [kind]` is the rest of the integer SIMD (`ir.VOP_*`): the
vector's bytes read as 8-, 16- or 32-bit lanes for wrapping and
saturating add and sub, multiplies, `pmaddwd`, `psadbw`, min, max,
compares, abs, 16-bit shifts, packs and zips (lane-moving ones within
each 128-bit half), as 64-bit lanes for `add64`, `sub64`, `shl64`,
`shr64` and `mulu32` (paddq, psubq, psllq, psrlq, and pmuludq: each
lane's product of its low 32 bits; std.core's `u64x2` and `u64x4`, which
Poly1305's AVX2 code uses), byte shifts of each 128-bit half by a constant
(`shrbytes`, `shlbytes`: psrldq, pslldq), `alignr` (palignr: the 16 bytes
of a pair from a constant byte on), the CPUs' SHA-256 instructions
(`sha256rnds2`, `sha256msg1`, `sha256msg2` of x86-64, with xmm0 as
sha256rnds2's third operand; `sha256h`, `sha256h2`, `sha256su0`,
`sha256su1` of arm64, which x86-64 targets refuse; scalarized where a
backend lacks them, as for the interpreter), the carry-less products of
two 64-bit lanes (`clmullo`, `clmulhi`: pclmulqdq 0x00 and 0x11, pmull
and pmull2), the CPUs' AES rounds (`aesenc`, `aesenclast` of x86-64's
AES-NI; `aese`, `aesmc` of arm64, which x86-64 targets refuse; scalarized
as the SHA ones), the moves between an `i64` and
an `i32x4`'s low 8 bytes (`from64`, `to64`: with a load or a store,
movq), and an `i32x8`'s 128-bit halves (`getlo`, `gethi`, `setlo`,
`sethi`, `join`); std.core's `u8x16`, `i16x8`, `i32x4`, `u64x2` and
their 32-byte forms are structs of four or eight `u32` words, so they share the
`i32x4`/`i32x8` types and `core.simdCast` between them is free. x86-64
selects one SSE2 or AVX2 instruction per `VOp`, or a short SSE2 sequence
for what 16-byte vectors would need SSE4.1 or SSSE3 for (`pmulld`,
`pminsd`, `pminud`, `pabsw`, `pabsd`, `psignw`, `psignd`, `phaddw`,
`phaddd`) and for unsigned compares; `pshufb` has no SSE2 equivalent, so
std's `u8x16.shuffle` is only called after `u8x16.ssse3()` (CPUID, which
std asks once per process for that and for AVX2).
`Clz`, `Ctz` and `Bswap` (of `i32` and `i64`) are `core.clz`, `core.ctz`
and `core.bswap`, behind the integers' `leadingZeros`, `trailingZeros`
and `swapBytes`: `bsr` or `bsf` with a `cmov` for 0, and `bswap`, on
x86-64 (whose baseline has no `lzcnt` or `tzcnt`), `clz`, `rbit` and
`clz`, and `rev` on arm64.
`CPUID [reg]` is a register of x86-64's `cpuid` (or `xgetbv`'s,
reg 4): `core.cpuid` and `core.xgetbv`, which tell whether the CPU has
AVX2 (the interpreter answers as one that does, with SSSE3). `Load`/`Store` move vectors
(unaligned), `VBuild` packs scalars, `VSplat` copies one to every lane and
`VExtract [lane]` reads one; they are never constants, call arguments or
results. An intrinsic reads its struct operands field by field into a
`VBuild` and writes its result back with `VExtract`s, so SROA still
splits the structs; instcombine then folds a `VBuild` of one vector's
lanes back into the vector and the four lane phis of a loop-carried
struct into one vector phi, which leaves chains of operations in vector
registers. The module has a text printer and parser, a verifier, dominator
and loop analyses, and an interpreter (`bic run --interp`, vectors
included) that serves as the oracle for optimizer tests.
`mir/scalarize.bi` turns vectors back into scalars for a backend without
SIMD lowering, and `i32x8`s and `VOp`s for the interpreter (a `VOp`
becomes i32 arithmetic on the words that hold its lanes, i64 arithmetic
on pairs of them for 64-bit lanes);
`mir/halve.bi` splits `i32x8`s into pairs of `i32x4`s for arm64's
16-byte registers.

## Optimizer (`compiler/mopt`)

`-O1` (and `-Og`) runs per-function cleanups (simplifycfg, SROA, mem2reg, instcombine,
SCCP, GVN, DSE, DCE). `-O2` (default) adds global optimization, bottom-up
cost-model inlining, devirtualization of closure calls, closure
specialization, loop simplification, LICM, rotation, full unrolling of
short loops, dead-loop deletion, strength reduction, bounds-check
elimination, if-conversion, switch formation and tail-recursion
elimination, iterated to a fixed point. `-O3` raises inlining and unrolling
budgets. `-Os` (`opt.Ctx.Size`) runs the `-O2` pipeline with small inlining
and unrolling budgets, no recursive self-inlining, no closure
specialization and no vectorization. Every pass is checked by the IR verifier and a differential fuzzer
that runs random programs through the interpreter before and after
optimization.

Passes that matter most for library-heavy code:

- **Closure specialization** (`specialize.bi`). A lambda is passed as a
  pointer to its function address and environment. A function that only
  reads such a parameter and passes it on (to itself, as a recursive sort
  does, or to other such functions) is cloned for each lambda its callers
  store there, so the indirect calls become direct and the lambda is
  inlined into the clone. This is what monomorphization over closure types
  gives Rust and C++; `List.sort` with `(a, b) => a < b` compiles to a sort
  with an inlined integer compare.
- **Nocapture inference** (`nocapture.bi`). A pointer parameter that the
  callee only loads, stores through, compares or passes to other nocapture
  parameters is nocapture (a pessimistic fixed point over the call graph).
  A stack slot whose address only reaches such parameters keeps precise
  alias facts: nothing not derived from it can point into it, and only
  calls that receive it can touch it. A `List` header on the stack stays in
  registers across heap stores and calls like `realloc`.
- **Dead-loop deletion** (`deadloop.bi`) removes loops with no observable
  effect whose exit test provably ends them, such as `List<bool>.drop`
  walking every element to drop nothing.
- **Trip counts** (`indvars.bi`). When a loop's single exit test compares
  an induction variable (a header phi stepping by a constant) with a
  constant, the iteration it leaves on is computed. Values that step by a
  constant are then replaced after the loop by their final values (a
  `List` filled by n pushes has length n afterwards, so the loops reading
  it lose their bounds checks); a test `x == c` that x can't reach while c
  only changes when the test holds is folded (`List.push`'s capacity check
  in a list made with room); and a loop left storing one byte to
  consecutive addresses becomes a `Memset`.
- **Bounds-check elimination** (`bce.bi`) proves comparisons from
  dominating branches, value ranges and induction over phis, including
  that sums and products of bounded non-negative values stay
  non-negative (`j = i * i; j += i`). Checks it cannot prove in a small
  innermost counted loop, whose index changes by a constant on every
  iteration (`c[i * n + j]` in a loop over `j`) and whose length is
  loop-invariant, are proved in the preheader instead by testing the
  first and the last index: the loop is versioned into a copy without
  them, entered when the test passes, and the original, which panics at
  the same iteration as before (matmul: 339 → 264 ms; at `-O2` and
  `-O3`, in the last per-function stage, not at `-Os`).
- **Private arguments.** The frontend marks the result slot and the owned
  copies of aggregate arguments `NoAlias`: nothing else the function
  receives points into them, so a function that doesn't let their
  address out treats them like its own stack slots (`Shared.new` sees the
  argument it moved out of is zero, and drops nothing).
- **Short cycles of small functions** (up to four) are inlined into each other (never a
  function into itself; `RecursiveInline` does that), so the four glue
  functions that drop a tree of `Shared` nodes become one recursive drop.
- **SLP vectorization** (`slp.bi`), last in the `-O2`/`-O3` pipeline on
  x86-64 (`opt.Ctx.vectorize`; the driver turns it off for arm64). Seeds
  are float stores to consecutive offsets from one pointer (four `f32`s
  become an `f32x4`, two an `f32x2`, two `f64`s an `f64x2`; a `vec3`'s
  x and y pack and z stays scalar), loop phis that start from such loads
  (a struct's fields promote keeps in registers across a loop: they
  become one vector phi, built before the loop, and lanes used only
  after the loop are extracted at its exit) and the two alike operands
  of one scalar op (the `x*x` and `y*y` of a dot product). From a seed the tree
  grows through ops that are the same in every lane; its leaves are
  consecutive loads (one vector load), one scalar in every lane (a splat),
  lanes of a vector made earlier, or a build. Nothing is reassociated, so
  results are bit-identical to the scalar code: a reduction is computed
  as vector lanes that are extracted and added in the original order.
  Vector loads and stores move only past memory operations alias analysis
  proves independent; a lane used before its vector exists stays scalar,
  and lanes used outside the tree are extracted (lane 0 is free). A cost
  model counts the instructions saved against the splats, builds and
  extracts added. `vec2`/`vec4` arithmetic, `mat4 * vec4` and n-body's
  velocity updates become `mulps`/`addps`/`subps` on whole vectors.
- **Loop vectorization** (`loopvec.bi`), just before SLP. An innermost
  one-block counted loop (as rotation leaves it; with bounds checks, the
  copy bce versioned) whose memory accesses are loads and stores of one
  float type at `base + es * i` (plus invariant terms) and whose floats
  are combined by lane-wise ops from those loads and loop-invariant
  scalars (`c[i * n + j] += aik * b[k * n + j]`, scaling, saxpy) gets a
  vector loop doing two `f64` or four `f32` per iteration, entered when
  at least that many iterations remain, with the original loop as the
  remainder and the fallback. Reductions and other loop-carried values,
  calls, other strides and values used after the loop leave it scalar.
  Two accesses whose byte distance `D` (constant, both advancing by `es`)
  is in `(0, es * VF)` would see each other's iterations reordered:
  with one base and a constant distance that rejects the loop
  (`a[i + 1] = a[i]`; `a[i] = a[i + 1]` and in-place updates are fine),
  otherwise the preheader tests it at run time and takes the scalar loop
  when lists overlap that way. Lanes compute what the scalar code did,
  so results are bit-identical (matmul: 270 → 154 ms).
- **Virtual calls** with at most six structs behind the receiver compare
  the vtable pointer with theirs and call each method directly (see
  [Inheritance](#inheritance-and-virtual-methods-cvirtbi-lvirtbi)):
  the branch needs one load instead of two, small methods are inlined,
  and conditional branches over a repeating mix of structs predict better
  than an indirect call.

Reference counts of strings and `Shared` are plain adds until std starts a
second thread (`threadStart` sets `rt.threaded` first); programs that never
start one have the test folded away and contain no `lock`-prefixed
instruction. Programs exporting C-callable functions set the flag at start.

## Backend (`compiler/mx86`, `compiler/marm64`, `compiler/melf`, `compiler/mpe`, `compiler/mmacho`, `compiler/mbe`)

Instruction selection fuses compares into branches and `cmov`s, folds
loads, immediates and address arithmetic into x86 operands, uses `lea` and
magic-number division, emits jump tables for dense switches and inlines
small memory copies. Vector values live in XMM registers (the float
class): `f32x2` loads and stores are `movsd`, 16-byte ones `movups`;
arithmetic is the packed SSE2 forms (`addps` .. `sqrtpd`, `xorps`/`andps`
with lane masks for negation and absolute value, `cmpps` for
`fmin`/`fmax`'s NaN rule), never with a memory operand; splats and builds
are `shufps`, `unpcklps` and `movlhps`, lane extraction `pshufd` (lane 0
is the register itself). A spilled vector gets a 16-byte slot. Baseline
SSE2 only, so the code runs on every x86-64, but for `i32x8`, which std
only reaches after CPUID says the CPU has AVX2: it lives in ymm
registers, with VEX-encoded three-operand AVX2 instructions (`vpaddd`
.. `vpxor`, the shifts, `vpshufb` for rotates by whole bytes,
`vpunpck*` and `vperm2i128` for the shuffles, `vpbroadcastd` for
splats; a 32-byte spill slot, `vmovdqu`). A function with `i32x8`
values copies and zeroes its vector registers with VEX forms (whole ymm
registers) and runs `vzeroupper` before its calls and returns, so SSE
code after it pays no transition. Registers are assigned by
iterated register coalescing (George & Appel) with loop-depth-weighted
spill costs.

Stack slots are shared (`mbe/slots.bi`, at `-O2`, `-O3` and `-Os`, for
both architectures): lowering gives every local and temporary its own
`Alloca`, and allocas whose contents are never needed at the same time
get one frame slot, as LLVM's StackColoring does. The IR has no lifetime
markers, so a slot holds a value where some path from the entry has
written it and some path onward reads it before overwriting all of it (a
`Memcpy`, `Memset` or run of stores that covers the slot before any read;
bytes that nothing in the function writes, such as a struct's padding,
count as covered, so a struct filled field by field in a loop is dead
around the back edge); a slot written where another holds a value, or an
instruction that reads one and writes the other, makes the two interfere,
and the slots are coloured greedily, largest first. A copy from one slot
into another after which the source is not read again doesn't make them
interfere: the two go in one class at the offsets that make the copy
move nothing (a temporary moved into a variable, a payload taken out of
a result, the fields of a tuple), as long as no member of the class is
written on bytes another holds a value in, and instruction selection
drops the copy. Only slots whose address stays
analysable take part: loaded from, stored to, compared, a memory
operation's operand, or passed to a nocapture parameter of a defined
function (recomputed over the final IR, optimistically through call-graph
cycles), which counts as a read and a write at the call. A slot whose
address is stored, passed to C, an indirect call or a capturing
parameter, merged by a phi or returned keeps its own space; so does every
slot of a function that calls `setjmp`, and with `-g` the slots of named
variables. Tasks are stackful, so an `await` is an ordinary call. The
register allocator shares spill slots the same way (as LLVM's
StackSlotColoring does): spill slots of registers never live together
get one, at the same optimization levels and not under `-g`, whose
variable locations name spill slots. Together this took `quic.newCore`'s
frame from 53.7 KB to 12.7 KB and `ssh.onKexEcdh`'s from 44.2 KB to
8.6 KB. A value moved out of a variable that is dropped later (the move
zeroes it, and the drop reads the zeroes) still keeps its own space.

The ELF writer emits a
PIE executable with its own `_start`, imports through the GOT (`BIND_NOW`,
RELRO) and a symbol table for profilers. Calls between Bismuth functions
follow the SysV x86-64 ABI on every OS, including struct-by-value C calls
in both directions on Linux and macOS.

An AArch64 backend (`--arch arm64`, Linux ELF for now) shares the MIR,
phi elimination and the register allocator; its encoder is
`compiler/marm64` and its selector and emitter `arm64_isel.bi`,
`arm64_call.bi` (AAPCS64 calls, C structs by value in both directions
included), `arm64_div.bi`, `arm64_simd.bi` and `arm64_emit.bi` in
`compiler/mbe`. Vectors are NEON (`arm64_simd.bi`, encoder
`marm64/neon.bi`): the vectorizers run for arm64 too, `i32x8`s become
pairs of `i32x4`s first (`mir/halve.bi`), and every `VOp` is one NEON
instruction or a short sequence; a function with a vector value it
doesn't cover would be scalarized. [arm64.md](arm64.md) has
the design; it writes ELF (Linux) and Mach-O (Apple Silicon), and
Windows on Arm is what it doesn't do yet.

### Targets

`backend.Options.os` picks the executable format; all three share the
code generator and differ only in the C calling convention and the writer,
which takes the same `elf.Program` (code, data, relocations, symbols):

| Target | Writer | C calls | Notes |
|---|---|---|---|
| `linux` | `compiler/melf` | SysV | PIE, `_start` calls `__libc_start_main`, links `libc.so.6`/`libm.so.6` |
| `windows` | `compiler/mpe` | Microsoft x64 (`win64.bi`) | PE32+ console or GUI image, ASLR with base relocations, a `.rsrc` section (manifest, optional icon and version information: `rsrc.bi`), imports from `ucrtbase.dll` (the Universal C Runtime) and the DLLs `@link` names; the IAT is the GOT; an entry stub calls `main` then ucrt's `exit`; stack probes for frames of 4 KB and more; 8 MB main stack |
| `darwin` | `compiler/mmacho` | SysV; Apple's AAPCS64 on arm64 (`arm64_call.bi`) | x86-64 or arm64 Mach-O PIE (arm64: 16 KB pages, `macho_arm64.bi`), `LC_MAIN` (dyld calls `main`), imports bound from `libSystem` through `__DATA_CONST,__got` with `LC_DYLD_INFO` opcodes, ad-hoc SHA-256 code signature (re-signed for an `.app` bundle with `--app`: `bundle.bi`), which Apple Silicon requires; static C objects and archives merged first (`machobj.bi`) |

On Windows only the boundary with C changes: calls to C pass arguments in
RCX, RDX, R8, R9 / XMM0-3 by position above a 32-byte home area, structs of
1, 2, 4 or 8 bytes as integers and others by reference to a copy, and
functions C calls (`main` and `export c fn`s, such as thread entry points)
receive them the same way, return a struct of 1, 2, 4 or 8 bytes in RAX
and others through the hidden pointer that comes in RCX, and also
preserve RDI, RSI and XMM6-15.

Calls through C function pointer types (`extern c fn(...) : R`) are
indirect calls (`OpCallInd`) with the same C signature as a direct
`extern c fn` call, so struct arguments and results follow the same rules
on each target. A COM method pointer (`extern com fn(*T, ...) : R`)
returning a struct on Windows is lowered as a call to the C function
`R* m(this, R* out, ...)`, MSVC's convention for member functions. `bic build --gui` sets the PE subsystem to GUI (no
console window).

The standard library has per-OS files picked by suffix (`wantsFile` in
`checker.bi`):
`name_linux.bi`, `name_darwin.bi`, `name_windows.bi`, and `name_unix.bi`
for both Linux and macOS. They hold what differs: raw I/O, errno, threads,
locks and condition variables, clocks, sleeping, thread-local storage, the
scheduler's stack switching and socket poller (`sched_*.bi`), file
flags and directory listing, and socket numbers and layouts. User
modules can do the same: when `m.bi` is missing, `import m` loads
`m_<os>.bi`, else `m_unix.bi` on Linux and macOS.

An `extern c fn` names its library with `@link("gfx")`, and `import c`
with `link "gfx"`. A name without an extension becomes the target's file
(`ir.libFile`): `libgfx.so` on Linux, with DT_RUNPATH `$ORIGIN` so the
executable finds it beside itself, `gfx.dll` on Windows (the DLL each
function is imported from), and `libgfx.dylib` on macOS.

`@link(static: "libgfx.a")` (a file, relative to the declaring source) and
`@link(static: "sqlite3")` (a C library bic builds, `cbuild.bi`) link C
code in instead, as does `--link-static file`: the driver collects the
static libraries of the imports left in the module (`drvlib.staticInputs`),
builds the bundled ones (cached), and `objlink.bi` merges the ELF objects
and archive members into the `elf.Program` before `elf.link`: sections,
symbol resolution between Bismuth, the objects and the shared libraries,
the objects' relocations as the Program's code, data and PC-relative data
relocations, and PLT stubs for calls to imported functions. For Windows
`mpe/coff.bi` does the same with COFF objects and archives before
`pe.linkDebug`, handing it what the Program cannot hold (`pe.Extra`: RVA,
section-relative and absolute-in-code relocations, the `.pdata` for the
exception directory, the TLS template and directory, static constructors
the entry stub calls, and the DLL of each import). See
[c-interop.md](c-interop.md#static-c-libraries).

## Shaders (`compiler/mshader`, `compiler/sbuild.bi`)

Functions marked `@vertex`, `@fragment` or `@compute` are also compiled
for GPUs ([shaders.md](shaders.md)). After checking, `checkShaders` builds
each stage and the functions it calls from their checked AST into a small
typed, structured shader IR (vector, matrix and struct values, member
accesses, `if`/`loop`/`break`), rejecting what the shader subset leaves
out, and three emitters turn it into SPIR-V 1.3, MSL 2.0 and HLSL (shader
model 5.0). `core.shader(f)` lowers to a constant `ShaderCode` with the
three and the stage's vertex and binding layouts; `bic shader` writes
them to files. The general SSA IR is not the input, because it has already
flattened vectors into byte offsets and control flow into an arbitrary CFG,
while SPIR-V needs typed access chains and structured control flow. The
CPU copies of the same functions go through the normal pipeline. Programs
without shader stages compile exactly as before.

## Self-hosting

bic is written in Bismuth and builds itself. `tools/ci/bootstrap.sh`
starts from a released bic, builds stage 1 from `compiler/`, has stage 1
build stage 2 and stage 2 build stage 3, and checks that stages 2 and 3
are byte-identical; `compiler/build.sh` is one such build. The
integration tests' `TestSelfHost` (`tests/rt_selfhost.bi`) checks the
same fixed point with the bic under test, at `-O0` too, and runs the
programs in `compiler/testdata` through stages 1 and 2. bic was first
written in Go; [compiler/README.md](../compiler/README.md) has the
history.

## Performance

`bench/run.sh` builds each benchmark in every profile, its C twin with
gcc and its Rust twin with rustc, and checks that the outputs match
(`QUICK=1` runs only the default build against `gcc -O2` and
`rustc -C opt-level=3`; `BASE=other/bic` adds another bic's default
build). The `bench` workflow runs it on the mesh's runner by hand, with
the `main` branch's bic as the base, and can print a benchmark's
optimized IR, machine code, compile time by phase
(`BIC_TIME=1`) and gcc's assembly for its C twin. Default-profile
numbers, x86-64 Linux, best of 5, all from one run, with bounds and
conversion checks on in bic and Rust:

| benchmark | bic (default, -O2) | gcc -O2 | rustc -O3 |
|---|---|---|---|
| fib(35), recursive | 11 ms | 13 ms | 20 ms |
| sieve, 50M, `List<bool>` with checked access | 255 ms | 249 ms | 252 ms |
| n-body, `vec3` library math | 208 ms | 211 ms | 153 ms |
| mandelbrot 1600×1200, f64 | 179 ms | 178 ms | 177 ms |
| sort 5M random ints (`List.sort`; C: `qsort`; Rust: `sort_unstable`) | 103 ms | 507 ms | 73 ms |
| 2M inserts + 4M lookups, `Map<int, int>` (C: open addressing; Rust: `HashMap`) | 206 ms | 177 ms | 241 ms |
| 2M interpolated strings, split and join | 45 ms | 49 ms | 63 ms |
| binary trees, 40 × depth 16, `Shared<Node>?` children (C: malloc/free; Rust: `Box`) | 62 ms | 78 ms | 75 ms |
| 20M virtual calls over 1000 components of 4 kinds (C: function-pointer tables; Rust: `Box<dyn Trait>`) | 21 ms | 107 ms | 94 ms |
| SHA-256 of 64 MB, then 100000 chained hashes of 32 bytes (`std.crypto`; C and Rust: portable round loops, no SHA instructions)* | 191 ms | 188 ms | 224 ms |
| matmul, 50 × 200×200 f64, i-k-j order, `List<f64>` with checked access† | 363 ms | 125 ms | 137 ms |
| particles, 4096 × 8000 steps, `List` of `vec3` position/velocity, gravity, damping, floor bounce† | 232 ms | 173 ms | 157 ms |
| spectral norm, n = 1500, f64† | 158 ms | 133 ms | 193 ms |
| ray tracer 384×384, 4 samples, 11 spheres and a plane, shadows, 4 bounces, `vec3`† | 109 ms | 204 ms | 60 ms |

\* From a later run on the same runner, with the x86-64 backend's `ror`
selection. Without it each rotate was a copy, two shifts and an or, and in
a local measurement the program took about 1.3 times as long as gcc's.

† Float-heavy benchmarks added later, measured in a local
`QUICK=1 RUNS=5` run on a different, noisier x86-64 Linux machine (4
cores), not on the runner, so they compare with each other but not with
the rows above. In that run n-body took 402 / 302 / 263 ms and
mandelbrot 351 / 331 / 333 ms. Their outputs match bit for bit across
bic, C and Rust, because each twin does its float operations in the same
order.

Timings on the runner vary by about 10% between runs. bic is ahead of
or level with gcc on nine of the ten rows measured there (the hash map is
16% behind) and ahead of Rust on six, level on two more. Rust's lead on n-body comes
from LLVM propagating the list length (5) from the pushes and vectorizing
the unrolled loops with `sqrtps`; on sorting from ipnsort's sorting
networks. Vector math written with library structs and operator methods
compiles to the same register code as hand-written C, because inlining,
SROA (including mixed-width slots), sret copy elision and scalar
promotion remove the temporaries.

What changed to get here, measured on the same benchmarks: closure
specialization and the introsort took sorting from 568 ms to 150 ms;
two-digits-at-a-time decimal formatting and non-atomic reference counts
took the string benchmark from 96 ms to 57 ms; nocapture-based alias facts
removed n-body's per-iteration reloads and bounds checks (231 → 200 ms);
dead-loop deletion removed a 50M-iteration empty loop from the sieve.
`Shared<T>?` and `Weak<T>?` take one word instead of two (null is the zero
pointer) and a `Shared` box's two counts share one word, which took binary
trees from 226 ms to 143 ms; printing skips the stdout lock until a second thread starts
(2M lines: 101 ms → 56 ms, `printf`: 70 ms).

With inheritance came the next round, against `main`'s bic in the same
run where it can build the program: trip counts (the sieve's fill loop
is a `memset` and its bounds checks are gone: 278 → 255 ms); `x!` of a
place lending instead of copying, private owned arguments and inlining
within short recursive cycles (binary trees 1.3× gcc's time before, now
ahead of both); one probe per `Map.set`, an unchecked probe start and
`grow` out of line (228 → 206 ms); and virtual calls of a few known
structs compared and called directly (21 ms, against 107 ms through
vtables).

The optimizer also has type-based alias analysis behind
`opt.Ctx.TypeBasedAlias`. It is off by default until the frontend formally
guarantees that memory is never reread with a different scalar class
outside `unsafe` punning.

## Optimization levels and build profiles

### Decision

**The default build is optimized and checked:** `-O2` with bounds,
division and conversion checks. This is Go's choice, and Zig's
ReleaseSafe. `bic run` and `bic build` share it. The named profiles are:

| flags | optimizer | frame pointers | checks | use |
|---|---|---|---|---|
| (none), `--release` | `-O2` | no | yes | everyday builds and shipping |
| `--debug` | `-Og` (= `-O1`) | yes | yes | stepping in a debugger: no inlining |
| `--unchecked` (with any profile) | as the profile | as the profile | **no** | best-case speed, measured first |

gcc's spellings adjust a profile: `-O0`..`-O3`, `-Os`, `-Og` (also `-O 2`
and `-O=2`; the last level given wins) and `-g`. `-g` adds debug
information (see [Debug information](#debug-information)) and keeps frame
pointers. `-O0` and `-O1` keep frame pointers too, because they cost
nothing measurable there. `bic version` prints what each profile means.

Why:

- **Optimizing costs little at compile time.** A sample program takes
  50–70 ms at `-O0` and 100–260 ms at `-O2`, so even an optimized rebuild
  finishes well within a second.
- **Unoptimized code is too slow for games.** n-body (`vec3` operator
  math, the shape of game code) runs 13 times slower at `-O0` and 5.6 times
  slower at `-Og` than at `-O2`, and 1.9 times slower at `-O0` than
  `gcc -O0`. Bismuth's library vectors, collections and operator methods
  depend on inlining far more than C does. A frame that takes 5 ms at
  `-O2` would take 65 ms at `-O0`, so a game wouldn't be playable, and
  that is why `-O0` can't be the default as it is in gcc.
- **`-O3` doesn't pay, so `--release` is `-O2`.** Compared with `-O2`, `-O3`
  compiles 2.7 times as slowly over the samples (up to 1.3 s for
  `collections.bi`), makes 20% larger executables and runs no faster. The
  sieve is 16% *slower* at `-O3`. Cargo's `--release` means opt-level 3,
  but here the default already is the release build. `--release` is
  accepted so cargo habits work, and so it can move to `-O3` if the larger
  budgets start to win. `-O3` stays available.
- **Checks stay on in release builds.** Unlike cargo's release, which
  drops overflow checks, every profile keeps them. `--unchecked` gains
  nothing measurable on these benchmarks, and Bismuth's integer
  arithmetic wraps anyway, so there are no overflow checks to drop.
  Dropping checks is a separate, explicit decision.

### Survey

| toolchain | default | debug | release / speed | size |
|---|---|---|---|---|
| gcc / clang | `-O0` | `-Og`, `-g`, `-fno-omit-frame-pointer` | `-O2`, `-O3` | `-Os`, `-Oz` (clang) |
| rustc / cargo | `dev`: opt-level 0, debug assertions, overflow checks | `dev` | `--release`: opt-level 3, no overflow checks | `opt-level = "s"/"z"` |
| Go | always optimized | `-gcflags=all="-N -l"` (no optimization, no inlining) | default | — |
| Zig | Debug | Debug | ReleaseSafe (checks), ReleaseFast (no checks) | ReleaseSmall |
| Odin | `-o:minimal` | `-o:none`, `-debug` | `-o:speed`, `-o:aggressive` | `-o:size` |
| Swift | `-Onone` (debug builds) | `-Onone`, `-g` | `-O`, `-Ounchecked` | `-Osize` |

Cargo profiles can also set `opt-level` per dependency, so a dev build can
optimize its libraries and leave the game's own code at `-O0`. bic compiles
the program and the standard library as one module, so there are no
separate dependencies to tune yet. Once there are packages, per-package levels
like cargo's `[profile.dev.package."*"]` would let the debug profile
optimize std and leave the program's own code easy to step through.

### Measurements

x86-64 Linux (4 cores). Compile time is a whole `bic build`, best of 5. Run
time is the best of 5. Size is the executable's.
These were measured when the levels were chosen, before closure
specialization, nocapture alias facts and non-atomic reference counts;
the current default-profile numbers are under [Performance](#performance).

| benchmark | -O0 | --debug (-Og -g) | default (-O2) | -O3 | -Os | -O2 --unchecked | gcc -O0 | gcc -O2 |
|---|---|---|---|---|---|---|---|---|
| fib run | 69 ms | 70 ms | 27 ms | 24 ms | 45 ms | 26 ms | 88 ms | 22 ms |
| sieve run | 1033 ms | 734 ms | 387 ms | 449 ms | 431 ms | 445 ms | 787 ms | 322 ms |
| n-body run | 4098 ms | 1718 ms | 308 ms | 304 ms | 309 ms | 304 ms | 2203 ms | 277 ms |
| n-body compile | 72 ms | 69 ms | 123 ms | 233 ms | 127 ms | 122 ms | 67 ms | 92 ms |

The ten samples run in 2–3 ms at every level, so only compile time and
size tell the levels apart:

| samples (10 programs) | -O0 | -Og | -O2 | -O3 | -Os |
|---|---|---|---|---|---|
| total compile time | 0.53 s | 0.67 s | 1.55 s | 4.18 s | 1.29 s |
| slowest compile (`collections.bi`) | 67 ms | 88 ms | 256 ms | 1255 ms | 199 ms |
| total executable size | 448 KB | 388 KB | 382 KB | 460 KB | 341 KB |

`-Os` makes executables 11% smaller than `-O2` and runs as fast as `-O2`,
except on fib, where it skips recursive self-inlining. The sieve's
timings vary from build to build by about 15% (code placement), which also
explains `--unchecked` coming out slower than checked.

### The backend at each level

The optimizer differs by level. In the backend only these vary:

- **frame pointers** at `-O0`, `-O1`/`-Og` and with `-g`. Keeping them at
  `-O2` costs about 1% (7% on call-bound fib), so they stay off there.
- **function alignment**: 16 bytes, or none at `-Os`.

Two other changes were measured and dropped. A faster register allocator
at `-O0` would save little: the whole backend takes about 9 ms per sample
program out of 50 ms at `-O0`. Aligning loop headers to 16 bytes at
`-O2` and above changed no benchmark by more than the noise.

### Debug information

`-g` and `--debug` add debug information; it is off by default. It is
never loaded at run time, so it costs only file size and compile time,
and it doesn't change the code: without `-g` executables are byte for
byte what they were, and with it only the debug sections (and frame
pointers, which `-g` asks for anyway) are added, and named variables
left in memory keep stack slots of their own (see the backend's slot
sharing).

| target | format | where |
|---|---|---|
| Linux (x86-64, arm64) | DWARF 4: `.debug_info`, `.debug_abbrev`, `.debug_line`, `.debug_loc`, `.debug_aranges` | non-allocated ELF sections |
| macOS | the same DWARF | a `__DWARF` segment before `__LINKEDIT` (not mapped readable, covered by the code signature); lldb reads it without a dSYM |
| Windows | CodeView in a PDB 7.0 file | `game.pdb` beside `game.exe`, named by the image's CodeView debug directory entry (RSDS: GUID, age, path) |

How positions get there:

- **Lowering** gives every IR value the position (file, line, column) of
  the statement or expression it comes from (`ir.Value.Pos`, a compact
  `ir.Pos` with a process-wide file table), each function its
  declaration's (`ir.Func.Pos`) and each block's control transfer its
  statement's (`ir.Block.Pos`: a `return`, a loop's back edge).
- **The optimizer** keeps positions when it rewrites a value in place,
  gives values it inserts the position of the value they replace
  (`insertBefore`), and the inliner keeps the callee's positions with an
  inlined-at chain (`ir.Func.Inlines`, `Pos.Inl`).
- **Instruction selection** puts the value's position on each MIR
  instruction (spill code takes the position of the instruction it
  serves), and the emitters bind a label wherever the line changes. Once
  jumps are relaxed the labels give the line table's addresses.

Functions are `DW_TAG_subprogram`s with their code range, and the generated
glue (drop, copy, format) is marked artificial. Variables and parameters
(`ir.Func.Vars`, recorded by lowering with `-g`) come with their types
(base types, pointers, structs with their fields, arrays, `str`, vectors)
and locations:

- a variable left in a stack slot is there for the whole function
  (`DW_OP_breg7 RSP+n`, `S_REGREL32`); with `-g` that slot is not shared
  with other allocas;
- a variable mem2reg promoted to SSA values is followed through register
  allocation: mem2reg records the values it is assigned
  (`ir.Func.DbgVals`), the allocator says which register or spill slot
  holds each, and a forward dataflow over the final machine code finds,
  at each instruction, which value the variable holds and whether its
  register still has it. The result is a location list (`.debug_loc`;
  `S_LOCAL` with `S_DEFRANGE_REGISTER`/`_REGISTER_REL` ranges in a PDB);
  constants are `DW_OP_implicit_value`;
- `ref` parameters and loop elements bound by reference are described
  through the pointer that holds their address.

At `-Og` (`--debug`) nothing is inlined and most values live in one
register, so gdb and lldb show every variable that is live. At `-O2 -g`
the line table is complete, but variables of inlined functions are not
shown and inlined code is attributed to the caller's frame (there are no
`DW_TAG_inlined_subroutine`s yet).

Standard library frames name their files `std/core/...` relative to the
build directory; point the debugger at a Bismuth checkout to see their
source (`directory`/`set substitute-path` in gdb, `settings set
target.source-map` in lldb).

The PDB holds the PDB info, TPI (types, with their hash stream), DBI
(module, section contributions and map, source files, the image's section
headers), IPI, `/names`, the module stream (`S_GPROC32`/`S_LPROC32` per
function with `S_FRAMEPROC`, the variables, and C13 line tables with the
file table), and public symbols (`S_PUB32`) with the globals and publics
hash streams. `bic build --target windows -g` writes it beside the
executable; `backend.compileDebug` returns it.

What is checked here (`tests/rt_debug.bi`): the line tables of
Linux x86-64 and arm64, macOS and Windows builds, against the test
runner's own DWARF reader (`tests/rt_dwarf.bi`), `llvm-dwarfdump` (with `--verify`) and `readelf`; gdb and
lldb breaking on a source line, stepping and printing arguments and
locals of a Linux build (the arm64 build was also stepped in lldb under
`qemu-aarch64 -g`); the Mach-O layout and code signature; and the PDB
with `llvm-pdbutil dump --all` (`llvm-symbolizer` also resolves addresses
of a Windows build to lines through it). Not run yet: lldb on a Mac, and
Visual Studio, WinDbg and RemedyBG on Windows.

Not done yet: `DW_TAG_inlined_subroutine` for inlined calls, lexical
scopes (all of a function's variables are at function level), variables
SROA splits into pieces (`DW_OP_piece`), call frame information
(`.debug_frame`/`.eh_frame`; debuggers unwind by the frame-pointer
prologue, which `-g` keeps), enum and closure types, and CodeView for
constants, `S_UDT` records and checksums of source files.

## Status

Implemented: everything in the spec's language sections. That covers types,
ranged/distinct types, generics, copy/ref/view passing, methods, packed
layouts with `comptime assert sizeof/offsetof`, enums/match, results and
nullables with `?`, `?.`, `??`, `!`, `defer`, `unsafe`, closures, `Shared`/`Weak`,
struct inheritance with virtual methods, `super`, `is` and `as`,
arenas, `spawn`/`await`/`all`/`race`/`timeout`, `parallel for` with chunk/tile
options and reductions, `Atomic`/`Mutex`/`Chan`, interpolated printing with
format specs, networking (framed TCP/UDP messages with reliable UDP and
lz4, byte streams, DNS, TLS 1.3 and 1.2 written in Bismuth, an
HTTP/1.1 client and server, WebSocket), C interop (`extern c fn`,
`bic bindgen`, `import c`), naming lints, and GPU shader stages
(`@vertex`, `@fragment`, `@compute`) compiled to SPIR-V, MSL and HLSL
([shaders.md](shaders.md)).

Platforms: bic runs on Linux, Windows and macOS (x86-64 and arm64) and
builds x86-64 executables for all three (`--target`) and arm64 Linux ones
(`--arch arm64`, the default on arm64 Linux). Where a machine can't run
the target's code, `bic run` uses the IR interpreter. C bindings
generated by `bic bindgen` build anywhere; `import c` needs a C preprocessor on the build machine.

Debug information (`-g`, `--debug`): DWARF line tables, functions and
variables for ELF (x86-64 and arm64) and Mach-O, and CodeView in a PDB for
Windows ([Debug information](#debug-information) has what is left).

Not implemented yet:

- **Native arm64 code for Windows** (Windows on Arm runs x64 emulated),
  waiting for hardware to test on. arm64 Linux and macOS are native, C
  structs by value included: see [arm64.md](arm64.md).
- **Hot reload** (`@hot` is accepted with a warning and compiles normally).
- In networking: `codec: .json` and `tickRate` (accepted and reserved),
  HTTP/2 and cookies ([stdlib.md](stdlib.md#networking)).
- Static C libraries are linked into Linux (x86-64 and arm64, ELF),
  Windows (x86-64, COFF) and macOS (arm64 and x86-64, Mach-O, without
  their unwind information) executables; not yet Windows on Arm
  ([c-interop.md](c-interop.md#static-c-libraries)).
  `import c "h.h" link "x.o"` still refuses object files and archives: use
  `@link(static: ...)` in bindings. Library search paths come with build
  profiles (#8).
- In C bindings: extern variables, bit-fields, unions with named members
  and typed function pointers (see [c-interop.md](c-interop.md#status)).
- Gamepads on macOS: `std.gamepad` returns `GamepadError.Unsupported`
  there until it has an IOKit HID backend ([stdlib.md](stdlib.md#gamepad)).
- Tasks run M:N on fibers (`std/core/sched.bi`): stackful, with a 1 MiB
  reserved stack each, switched with `_setjmp`/`_longjmp` on Linux and
  macOS and with the OS's fibers on Windows; worker threads per core with
  work stealing (those started beyond the core count for blocked ones end
  after 10 s idle), one socket poller per process, and a monitor thread that
  hands off work queued behind a task blocking its thread and, while tasks
  wait for sockets and no worker is in the poller, polls them itself every
  100 us. Tasks are not preempted: loops that may find their input ready
  every time (the ssh reader, the SFTP server) call `core.yieldIfLong()`
  per message, which yields once the task has run 2 ms. `parallel for`
  runs on a pool of its own (`Pool` in `std/core/task.bi`, as many threads
  as the scheduler has workers), not on the scheduler's workers, with the
  thread that reached it taking jobs too. Cancellation is cooperative: blocking std
  operations (sleep, channels, locks, sockets) check it and wake for it.
  The IR interpreter (`--interp`) does not run tasks.

Design decisions taken where the spec was open:

- Vectors, matrices, `quat`, `Angle`, `Duration`, `Shared`, `Task`, collections
  and all concurrency types are **library types** (`std/core`), built from a
  small native core. This was requested explicitly: nonstandard types and
  functionality belong in the library.
- Untyped float literals default to `float` (`f32`); integer literals to
  `int` (`i64`). Integer arithmetic wraps; conversions `u8(x)` and ranged
  assignments are checked; `T.wrap/clamp/check(x)` are the explicit
  alternatives.
- `--unchecked` removes bounds, division and conversion checks for
  best-case C speed.
