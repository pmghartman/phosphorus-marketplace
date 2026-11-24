# C interop: reading headers, bindings and linking

Issue #7 asked whether `import c "gfx.h" as rl link "libgfx.so";` is the
right model for calling C from Bismuth, or an odd mix of source and build
configuration. This document compares how other languages do it, gives
the model Bismuth adopts and why, and records what is done and what comes
next.

## Where Bismuth was

`import c` ran the build machine's C preprocessor (`cc -E`) on every
compile, parsed the declarations, and mapped C types one to one. That made
the one-line import pleasant, and caused these problems:

- **No C compiler, no build.** Every build of a program that imports a
  header needs `cc`. Most Windows machines have none.
- **The host's header, not the target's.** The header was preprocessed
  with the build machine's macros, and its types were laid out with Linux
  x86-64 sizes for every target. `long` was 8 bytes under
  `--target windows`, where it is 4 (LLP64), so any struct or function
  using `long` had the wrong layout or signature on Windows.
- **Library files in source.** `link "libgfx.so"` names a Linux file. The
  same source could not link `gfx.dll` or `libgfx.dylib`, and on Windows
  the header's functions were imported from the C runtime instead of the
  library.
- **Not reproducible.** What a build sees depends on the machine's
  compiler and system headers, and the language server ran the
  preprocessor on each analysis.

The standard library never used `import c`. It declares `extern c fn`s by
hand, with `@cname` and `@link("kernel32")`.

## How others do it

| Language | Reads C headers when | Declarations live in | Library named in | Needs a C toolchain to build |
|---|---|---|---|---|
| Zig | every compile (`@cImport`, translate-c on a bundled clang) | the header | `build.zig` (`linkSystemLibrary`), or `extern "lib"` in source | no: Zig ships clang and libc headers for its targets |
| Rust | at generation time (bindgen on libclang), in `build.rs` or ahead of time | generated Rust, often checked in (`-sys` crates, the `libc` crate) | source (`#[link(name = "foo")]`) or `build.rs` (`cargo:rustc-link-lib`); search paths in `build.rs` | no, with checked-in bindings |
| Go (cgo) | every build (the comment preamble is C) | the preamble | `#cgo LDFLAGS: -lfoo`, per OS with `#cgo windows LDFLAGS:` | yes, for the target. cgo is why cross-compiling Go is easy only without C |
| Swift | every compile (the Clang importer is in the compiler) | the header, found through a module map | the module map (`link "foo"`), or SwiftPM `systemLibrary` | no extra one: Swift ships clang |
| Odin | never | hand-written `foreign` blocks (the `vendor:` collection ships them) | source: `foreign import lib "system:foo"`, chosen per OS with `when ODIN_OS == ...` | no |
| Jai | at generation time (the `Bindings_Generator` module uses libclang in a metaprogram) | generated Jai, one file per OS, checked in | source: `#foreign_library "foo"`, the extension added per platform | no |
| Nim | every compile, by the C compiler Nim emits code for | `{.importc, header: "foo.h".}` pragmas, or generated (c2nim, Futhark) | `{.dynlib: "libfoo.so".}`, picked per OS with `when defined(windows)` | yes: Nim compiles to C |
| C# | never (ClangSharp's generator, run by hand) | `[DllImport]` / `[LibraryImport]` declarations, checked in | source: `[DllImport("foo")]`. The runtime tries `foo.dll`, `libfoo.so` and `libfoo.dylib`, and `NativeLibrary.SetDllImportResolver` overrides it | no |

The pattern is clear:

- **Reading headers at every compile works when the compiler carries its
  own C front end and target headers.** Zig and Swift bundle clang; Nim and
  cgo use the C compiler they need anyway. Zig has accepted moving
  `@cImport` out of the language into the build system (translate-c as a
  build step), because the import ties every compile to a C front end and
  makes caching and tooling harder. bic has no C front end of its own,
  so it borrowed the build machine's: the worst case of the model.
- **Languages that build without a C toolchain use declarations in their
  own syntax**, generated or hand-written, and checked in: Rust's `-sys`
  crates, Odin, Jai and C#. Generation runs once, on a machine with clang
  or cc, and the output is reviewed like code.
- **The library's name is in source; where to find it, and how to link
  it, is build configuration.** Rust's `#[link(name = "foo")]`, Odin's
  `"system:foo"`, Jai's `#foreign_library "foo"` and C#'s
  `DllImport("foo")` all name the library without its platform's prefix
  and extension. Search paths, static or dynamic linking, and versioned
  file names come from `build.rs`, the build file or the command line.
  Which library provides `InitWindow` is a fact about the binding, like
  its signature. Where `libgfx.so` sits on this machine is not.
- **Bindings are per target.** A header with `long`, `#ifdef _WIN32` or a
  platform typedef produces different declarations for each target.
  bindgen output is target-specific, Jai generates one file per OS, and
  the Rust `libc` crate is written per target.

## The model for Bismuth

1. **Bindings are ordinary Bismuth source.**
   `bic bindgen gfx.h -o gfx.bi` writes the header's functions, structs,
   enums and constants as `extern c fn`, `struct`, `type` and `const`
   declarations. The output is checked in and imported like any other
   module (`import gfx as rl;`). Building needs no C compiler and gives
   the same result on every machine. Generating does need one; that runs
   once, when the header changes.
2. **Bindings are generated for every target.** bindgen preprocesses the
   header as each target's compiler would (its OS macros, Windows's LLP64
   sizes) and compares the three results. When they agree, it writes one
   file. When they differ, it writes `gfx_linux.bi`, `gfx_windows.bi` and
   `gfx_darwin.bi`, and `import gfx` picks the target's file, as the
   standard library picks its per-OS files.
3. **Source names the library, not its file.** `@link("gfx")` links
   `libgfx.so` on Linux, `gfx.dll` on Windows and `libgfx.dylib` on macOS.
   A Linux executable looks beside itself first (RUNPATH `$ORIGIN`), as a
   Windows one does, and `bic run` adds the program's directory to the
   loader's search path. Search paths, static linking and per-machine
   overrides belong on the command line and in the project manifest that
   build profiles (#8) introduce, not in `.bi` files.
4. **`import c` stays as a convenience for exploring a header.** It shares
   bindgen's preprocessor run and parser, so it now uses the target's type
   sizes and macros. It accepts only a bare library name or the target's
   own kind of file. Without a C compiler, it explains how to generate
   bindings instead. It still needs a C compiler on every build machine
   and reads the build machine's system headers, so programs meant for
   others should check in generated bindings.

Why this over the alternatives:

- **Correct cross builds.** A Windows binding is generated for Windows. It
  does not reuse the host's layout. `bic build --target windows` on Linux
  needs nothing the Windows machine would not.
- **Hermetic and cacheable.** Inputs are `.bi` files only. There is no
  dependence on `cc`, system headers or `CC`/`CFLAGS`, so the build
  output depends on the source alone. That also simplifies a future
  compile cache.
- **Readable and reviewable.** The declarations are in the repository. A
  header update shows up as a diff of signatures and layouts, the same in
  code review as in `git blame`.
- **Editor tooling for free.** bils sees ordinary declarations: hover,
  go-to-definition and completion work on C functions and structs without
  running a preprocessor on every keystroke.
- **Fits the language.** The standard library already binds C this way
  (`extern c fn` with `@cname` and `@link`). Generated bindings use the
  same mechanism, so there is one path into C, and `import c` is a
  shortcut to it.
- **Bundling a C front end, as Zig does, is out of reach.** It would make
  bic depend on clang, or require writing a C preprocessor and parser plus
  target system headers. Reading headers at every compile without one is
  the model that broke.
- **Keeping only `import c` keeps every problem above.** A cache would
  speed it up, but builds would still need a C compiler and the host's
  headers.

## What generated bindings look like

```bismuth
// Code generated by bic bindgen from gfx.h; DO NOT EDIT.
//
//   bic bindgen gfx.h -o gfx.bi
//
// The same for Linux, Windows and macOS. The functions come from the
// library "gfx": libgfx.so, gfx.dll or libgfx.dylib.

struct Color {
    u8 r;
    u8 g;
    u8 b;
    u8 a;
}

const RED = Color { r: 230, g: 41, b: 55, a: 255 };   // #define RED (Color){ 230, 41, 55, 255 }
const MAX_FRAMES = 3;

@link("gfx") extern c fn InitWindow(i32 width, i32 height, str title);
@link("gfx") extern c fn LoadDefaultTexture() : Texture;
```

and, from a header whose function returns a pointer:

```bismuth
@link("widgets") unsafe extern c fn widget_new(str name, Shape shape) : *Widget;
```

The rules:

- Only declarations from the header's own directory are emitted, not the
  system headers it includes.
- C types map by size on the target. `long` is `i64` on Linux and macOS
  and `i32` on Windows. `size_t` is `u64`, and fixed-width typedefs map to
  their Bismuth types.
- A `const char*` parameter is a `str`, passed as a pointer to its
  NUL-terminated bytes.
- Functions that take or return other raw pointers are `unsafe extern c
  fn`, callable only inside `unsafe { }`, unless the header marks them
  `@bi.safe`.
- C enums are `type E = i32;` plus `const i32` constants.
- Object-like macros become constants. That covers numbers (with their
  `u`/`l`/`f` suffixes), strings, integer expressions of other constants,
  and compound literals of a struct (`(Color){...}`, designated or not),
  which become struct constants. Integer expressions are evaluated with
  C's types: `UINT64_C(...)`, `(uint32_t)` casts, `-1u` and the operators
  `| & ^ ~ << >> + - * / %`, comparisons and `?:`; a macro written with a
  function-like macro (`#define E (D(3))`) is read as the preprocessor
  expands it. An unsigned or 64-bit result is typed (`const u64`,
  `const u32`, `const i64`); an `int` one is an untyped constant.
- Opaque structs become empty structs, to use through pointers.
- Unions become a byte array of the union's size and alignment, with a
  comment naming the members.
- Function pointers are `*void`.
- What has no Bismuth form yet is left out with a note on stderr: extern
  variables, bit-fields, `static inline` functions (no library exports
  them), function-like macros and `va_list`.
- The first line marks the file as generated, so the compiler does not
  lint its C-style names.

## Status

Done (#7):

- `bic bindgen` with per-target generation and per-target files
  (`m_<os>.bi`, `m_unix.bi`) picked by `import`.
- Library names by target (`@link("gfx")`, `link "gfx"`) with RUNPATH
  `$ORIGIN` on Linux, and `bic run` finding libraries beside the program.
- `import c` with the target's type sizes and OS macros, the target's
  library on Windows and macOS, errors for another OS's library file or a
  static archive, and a clear error without a C compiler.
- Language support the bindings need: struct-valued constants, a `str`
  argument to an `extern c fn` passed as `const char*`, and no naming
  lints in generated files.
- `examples/particles` builds for Linux, Windows and macOS from its
  checked-in `gfx.bi`, with no C compiler.

Done (#20), structs by value in both directions:

- An `export c fn` takes structs by value and returns structs with the
  target's C convention on every x86-64 target. On Windows (Microsoft
  x64), structs of 1, 2, 4 or 8 bytes arrive in an integer register or
  stack slot, floats included, and others as a pointer to the caller's
  copy; a result of 1, 2, 4 or 8 bytes goes back in RAX, and others
  through the hidden pointer that comes in RCX ahead of the arguments
  and goes back in RAX. Linux and macOS use SysV's classification. Its
  parameters and result cannot own memory (`str`, `List<T>`), as C
  copies them bitwise, and a direct call from Bismuth uses the same
  convention as C's.
- COM method pointer types, `extern com fn(*T, ...) : R`. MSVC returns
  every struct from a C++ member function through a hidden pointer
  passed after `this` (and back in RAX), even one a C function would
  return in RAX, such as D3D12's 8-byte `D3D12_CPU_DESCRIPTOR_HANDLE`.
  Elsewhere (the Itanium C++ ABI) a method returns as a C function does,
  so the type is called as `extern c fn` there.
- `tests/rt_cabi.bi` checks both conventions against a C++ compiler:
  a library with struct-valued callbacks, functions and a virtual class,
  built with the system's `c++` on Linux, and with clang for
  `x86_64-pc-windows-msvc` and lld-link for Windows, where the program
  runs under Wine. `tests/run/cabi_windows.bi` runs in Windows CI and
  checks each struct size class against the convention's scalar form.

Done (#78), static C libraries: see [Static C libraries](#static-c-libraries)
below.

Next:

- **Search paths in the build configuration** (with #8): `-L`-style
  directories, and a library's file per target when it isn't
  `lib<name>.so`.
- **A per-module library default**, so a file of bindings can say
  `@link("gfx")` once instead of on every function.
- **More of C:** extern variables, unions with named members, typed
  function pointers, and bit-fields.
- **`bic bindgen --check`** for CI: regenerate and fail if the checked-in
  bindings are stale (`tests/rt_cinterop.bi` does this for particles).
- **Decide later whether `import c` stays.** Once bindings cover what
  real headers need, `import c` could be reduced to `bic bindgen` run at
  build time, or deprecated.

## Static C libraries

Issue #78 asked for programs that need no SQLite or OpenH264 shared
library on the machine they run on, as Rust's `rusqlite` with its
`bundled` feature and `openh264-sys2` give. bic links static C code into
Linux, Windows (x86-64) and macOS executables itself: it has no other
linker to call, and its ELF, PE and Mach-O writers (`compiler/melf`,
`compiler/mpe`, `compiler/mmacho`) were already small static linkers for
Bismuth's own code.

### In source and on the command line

```bismuth
// a file of bindings beside libgfx.a (or gfx.o)
@link(static: "libgfx.a") extern c fn InitWindow(i32 width, i32 height, str title);

// a C library bic builds itself (third_party/sqlite)
@link(static: "sqlite3") unsafe extern c fn sqlite3_libversion() : *u8;
```

```bash
bic build --link-static vendor/libgfx.a --link-static extra.o game.bi
```

- **`@link(static: "path")`** names an archive (`.a`, `.lib`) or object
  (`.o`, `.obj`) file, relative to the file that declares the function. A
  path without an extension is the target's archive in that directory:
  `lib<name>.a` (`<name>.lib` on Windows). The functions are linked into
  the executable, and no `DT_NEEDED` entry or DLL import is written for
  them.
- **`@link(static: "name")`**, a bare name, is a C library bic bundles and
  builds: `sqlite3` and `openh264` (the recipes in `compiler/cbuild.bi`).
- **`--link-static file`** (repeatable) adds an archive or object to the
  link without changing the source, for functions declared without
  `@link`, as C library functions are. This follows the model above:
  where a library comes from on one machine is build configuration.

A static library is linked only when the program uses one of its
functions, as an imported shared library function is: the IR holds the
functions that are called (or whose address is taken with `core.fnAddr`),
so a module can declare static bindings that cost nothing until used.
That is how `std.db.sqlite` offers its bundled SQLite: `sqlite.useBundled()`
takes the addresses of the static functions, so calling it is what links
SQLite in, and programs that don't keep loading the system's library at
run time.

### The linker (`compiler/melf/objlink.bi`)

objlink reads ELF64 relocatable objects and `ar` archives (GNU and BSD
member names) for x86-64 and AArch64, and merges them into the
`elf.Program` that bic made of the Bismuth code, before `elf.link` lays out
the image:

- **Sections.** Code goes to `.text`, read-only data to `.rodata` (or to
  the RELRO data when it holds absolute pointers, which need dynamic
  relocations in a PIE), `.data.rel.ro*` to RELRO, writable data to `.data`
  and `NOBITS` to `.bss`, each at its alignment. Notes, `.eh_frame` and the
  sections of a COMDAT group already taken (C++ inline functions) are left
  out. Debug sections are not kept.
- **Symbols.** Bismuth's functions and globals, the objects' definitions
  (strong over weak, common symbols merged into `.bss`) and the archive
  members resolve against each other. A member is linked when it defines
  a symbol something linked needs, repeating until nothing changes (ld's
  `--start-group`), so archive order does not matter and unused members
  stay out. What nothing defines is imported from the shared libraries the
  program already links (libc, libm), weakly for a weak reference (null
  when missing). C can call `export c fn`s by name.
- **Relocations.** x86-64: `PC32`, `PLT32`, `GOTPCREL`, `GOTPCRELX`,
  `REX_GOTPCRELX` in code; `64` (absolute pointers, as `RELATIVE` or
  symbol dynamic relocations), `PC32` and `PC64` (switch tables) in data.
  AArch64: `ADR_PREL_PG_HI21(_NC)`, `ADR_PREL_LO21`, `ADD_ABS_LO12_NC`,
  `LDST8/16/32/64/128_ABS_LO12_NC`, `CALL26`, `JUMP26`, `CONDBR19`,
  `TSTBR14`, `ADR_GOT_PAGE`, `LD64_GOT_LO12_NC` in code; `ABS64`, `PREL32`
  and `PREL64` in data. GOT references share Bismuth's GOT. A call to an
  imported function gets a PLT stub (`jmp *GOT(f)(%rip)`; `adrp`/`ldr`/`br
  x17` on AArch64).

The code must be position independent, as every executable bic writes is
a PIE: build it with `cc -fPIC`. An absolute address in code (`R_X86_64_32`,
`MOVW_UABS`), or a PC-relative reference to a variable a shared library
defines (`-fPIE` or `-fno-pic` code reading `stderr`, which needs a copy
relocation), is an error that says so. So are thread-local variables
(`.tdata`, `.tbss`), static constructors (`.init_array`), `STT_GNU_IFUNC`
definitions, `REL` relocations and LTO (bitcode) objects. None of SQLite,
OpenH264's encoder or plain C libraries built with `-fPIC` need them.

### The Windows linker (`compiler/mpe/coff.bi`)

coff reads COFF objects for x86-64 (`.obj`, or `.o` from mingw-w64's gcc,
clang and zig; `/bigobj` ones too) and archives of them (`.lib` from
`lib.exe` or `llvm-lib`, `.a` from `ar`: `!<arch>` with the linker members
and MSVC's or GNU's long names), and merges them into the `elf.Program`
before `pe.linkDebug` writes the image. What the Program has no form for
goes to a `pe.Extra`:

- **Sections.** Code goes to `.text`, read-only data to `.rdata` (or to the
  writable data when it holds absolute pointers, which take base
  relocations), writable data to `.data`, uninitialized data to `.bss`, and
  `.tls$` to the thread-local template (the image's `.tls` and TLS
  directory, with `_tls_index`). The objects' `.pdata` is gathered into the
  image's `.pdata`, sorted, as the exception directory, so Windows and
  debuggers unwind through C frames; `.xdata` stays read-only data. A COMDAT
  section (C++ inline functions and vtables, mingw's `.refptr.`) is taken
  from the first object with its symbol (or, for gcc's `.pdata$`/`.xdata$`,
  its name), and its associative sections go with it. `.drectve`, debug
  and other removable or discardable sections are left out.
- **Symbols.** As on Linux, archive members are taken while they define a
  symbol still needed, strong definitions win over weak and common ones,
  and a weak external takes its default when nothing else defines it (an
  undefined weak function's address is null; a call to it, which code
  makes only after checking that, traps). C can call `export c fn`s.
- **Imports.** A symbol nothing defines is imported from a DLL, through
  the IAT that Bismuth's own imports use: `__imp_X` (a `dllimport`
  declaration) is X's IAT slot, and a direct call or a function pointer
  goes through a stub, `jmp *IAT(X)(%rip)`. The DLL is the one the Bismuth
  program imports X from, else the one an import library among the inputs
  names (MSVC's short import members, as `llvm-dlltool` and `lib.exe` make
  them, and mingw-w64's dlltool members: pass `kernel32.lib` or
  `libpsapi.a` with `--link-static`), else `ucrtbase.dll` (the C runtime
  the Bismuth runtime uses too) for a lower-case or underscore name and
  `kernel32.dll` for a Win32 one, with a table of exceptions (winsock to
  `ws2_32.dll`, `Reg*` to `advapi32.dll`, `Nt*` to `ntdll.dll` and so on).
  A GNU C++ name (`_Z...`) that nothing defines and no import library
  names is a link error listing each such name and the object that needs
  it, since no system DLL exports one and the program could not start.
- **The compilers' static runtimes.** bic links no `libgcc`,
  `libmingwex`, `libucrt.a` or MSVC CRT, so it makes what C code expects
  from them when it is needed: the stack probes `___chkstk_ms` and
  `__chkstk`, the printf family (`printf`, `fprintf`, `sprintf`,
  `snprintf`, `sscanf` and their `v` forms, mingw's `__mingw_` ones and
  MSVC's `_snprintf`) over the UCRT's `__stdio_common_` functions, which
  `ucrtbase.dll` exports, `_fltused`, the `/GS` cookie and `_tls_index`.
- **Relocations.** `IMAGE_REL_AMD64_REL32` and `REL32_1` to `_5` in code
  and data (switch tables), `ADDR64` (base relocations, in code too, for
  `-mcmodel=large`), `ADDR32NB` (`.pdata`, `.xdata` and
  `__ImageBase`-relative tables), `SECREL` (thread-local variables) and
  `SECTION`. `__ImageBase` and `__image_base__` are the image's base.
- **Constructors.** `.ctors` (gcc, clang for mingw) and `.CRT$XI*`,
  `.CRT$XC*` (MSVC) run from the entry stub before `main`; `.CRT$XL*` are
  TLS callbacks.

Errors say what is not supported: `ADDR32` (a 32-bit absolute address),
static destructors in `.dtors` and terminators (`.CRT$XP*`, `.CRT$XT*`),
imports by ordinal, LTO (bitcode) members, objects for another machine,
gcc's emulated thread-locals (`__emutls_get_address`, in libgcc: build with
clang or zig, whose thread-locals are native) and libgcc's other helpers.
A variable a DLL exports must be declared `__declspec(dllimport)`:
mingw's auto-import (pseudo-relocations) is not done. C++ objects must
use the GNU (Itanium) C++ ABI of mingw-w64 targets; MSVC-ABI C++ needs its
runtime, which bic does not provide.

### The Mach-O linker (`compiler/mmacho/machobj.bi`)

machobj does the same for macOS: it reads 64-bit Mach-O relocatable
objects (`MH_OBJECT`) and `ar` archives of them for x86-64 and arm64, and
merges them into the `elf.Program` before `macho.link` lays out the image.

- **Archives.** BSD archives (`#1/<n>` long member names, the
  `__.SYMDEF` and `__.SYMDEF SORTED` symbol tables and their `_64` forms,
  as `ar`, `libtool` and `llvm-ar --format=darwin` write them) and GNU
  ones. A member is taken when the archive's symbol table says it defines
  a symbol still undefined (its own symbol table, for an archive without
  one), repeating until nothing changes.
- **Sections.** Code (`__TEXT,__text` and any section of instructions)
  goes to `__TEXT,__text`; `__TEXT`'s `__const`, `__cstring` and
  `__literal4/8/16` to `__TEXT,__const`, or to `__DATA_CONST,__const`
  when they hold absolute pointers; `__DATA,__const` and `__DATA_CONST` to
  `__DATA_CONST,__const` (read-only after dyld's fixups); other data to
  `__DATA,__data`; zero-fill sections (`__bss`, `__common`) and common
  symbols to `__DATA,__bss`. Debug sections and `__LLVM` are dropped, and
  so is unwind information (`__LD,__compact_unwind`, `__TEXT,__eh_frame`):
  the writer emits no `__unwind_info` for Bismuth's code either, so the C
  frames have none. That costs nothing for C, but a C++ exception cannot
  cross them (OpenH264 is built with `-fno-exceptions`), and debuggers and
  crash reports unwind them by frame pointer only.
- **Symbols.** C's names carry a leading underscore and Bismuth's do not:
  `_sqlite3_open_v2` in an object is Bismuth's `sqlite3_open_v2`, and an
  undefined `_malloc` is an import of `malloc`. Weak definitions
  (`N_WEAK_DEF`: C++ inline functions and templates) yield to strong ones,
  the largest common symbol wins, and what nothing defines is imported from
  libSystem (libc, libm, pthreads and the compiler's arithmetic helpers are
  all there), weakly for `N_WEAK_REF`. `export c fn`s are callable from C.
- **Relocations.** arm64: `ARM64_RELOC_UNSIGNED`, `BRANCH26`, `PAGE21`,
  `PAGEOFF12` (an `add` or a load or store, scaled by its size),
  `GOT_LOAD_PAGE21`/`GOT_LOAD_PAGEOFF12`, `POINTER_TO_GOT`, `SUBTRACTOR`
  and `ADDEND`. x86-64: `X86_64_RELOC_UNSIGNED`, `SIGNED`, `SIGNED_1`,
  `SIGNED_2`, `SIGNED_4`, `BRANCH`, `GOT_LOAD` (rewritten from `movq` to
  `leaq` for a symbol defined in the program), `GOT` and `SUBTRACTOR`.
  GOT references go through `__DATA_CONST,__got`: rebased slots for
  symbols defined in the program, bound ones for imports. A call to an
  imported function gets a stub that jumps through its slot. Absolute
  pointers in data are rebased or bound (with their addend); a difference
  of two addresses (`SUBTRACTOR`) is a constant when both are in one output
  section and a PC-relative value when the subtracted one is in the
  section that holds it (switch tables).

The ad-hoc code signature is computed over the finished image as before,
so arm64 executables with C in them stay signed. Thread-local variables
(`__thread_vars`), static constructors (`__mod_init_func`), arm64e
pointer authentication, universal (fat) files and LTO (bitcode) objects
are errors that say so.

### Bundled libraries (`compiler/cbuild.bi`)

A recipe names a directory under `third_party/`, its sources and its
flags. bic compiles each source with the build machine's C compiler (`$CC`,
else `cc`; `$CXX` or `c++` for C++), with `-fPIC`, up to one compiler per
CPU at a time, and keeps the objects in a cache keyed by the SHA-256 of the
recipe, the compilers (and their `--version`), the flags and every file of
the source tree: `$BIC_CACHE`, else `$XDG_CACHE_HOME/bic` or `~/.cache/bic`
(`~/Library/Caches/bic`, `%LOCALAPPDATA%\bic\cache`). A first build says
what it compiles; later ones reuse the objects. Building for another
architecture takes `<triple>-gcc` or `clang --target=<triple>`. The objects
are linked like an archive's members, so only what the program needs goes
in.

For Windows (x86-64, whether bic runs on Windows or cross-compiles from
Linux or macOS) the compiler must target `x86_64-windows-gnu`, mingw-w64's
headers and the GNU C++ ABI: `$BIC_WINDOWS_CC` (`$BIC_WINDOWS_CXX` for
C++) when set, on Windows `$CC` (`$CXX`), else the first found of
mingw-w64's `x86_64-w64-mingw32-gcc` (`-g++`), `zig cc` (`zig c++`) with
`-target x86_64-windows-gnu`, on Windows `gcc` (`g++`) as MSYS2's mingw64
and ucrt64 environments have it, and `clang --target=x86_64-w64-windows-gnu`
(`clang++`), which needs mingw-w64's headers (llvm-mingw has them, as does
a Linux with the `mingw-w64` package; off Windows a clang that can't find
`stdio.h` for that target is passed over, and the build says there is no C
compiler for Windows). zig is the easiest anywhere: one
download, with its own headers. The sources are compiled with
`-D__MSVCRT_VERSION__=0xE00 -D__USE_MINGW_ANSI_STDIO=0` (the UCRT, which
the Bismuth runtime uses, and its printf rather than mingw's) and `-g0`,
keeping the unwind tables; the objects are cached apart from Linux's. All
three were checked against SQLite and OpenH264: the executables link, and
their relocations, imports and unwind tables were compared with the
objects.

For macOS (`--target macos`, either `--arch`), the objects are Mach-O and
need a compiler that targets macOS with its headers: `$BIC_MACOS_CC`
(`$BIC_MACOS_CXX` for C++) when set, a whole command such as
`clang --target=arm64-apple-macos11 -isysroot /path/MacOSX.sdk`; else, on a
Mac, `$CC` or `cc` (Xcode's command line tools) with `-arch arm64` or
`-arch x86_64`; else, on Linux or Windows, `clang --target=<arch>-apple-macos`
with the SDK `$SDKROOT` names, or `zig cc -target aarch64-macos` /
`x86_64-macos` (zig ships macOS's libc headers, so it needs no SDK). A
plain clang without an SDK is not used: it would compile against the
host's glibc headers, whose `errno`, `stat` and the like do not match
libSystem's. The release builds of bic for macOS run on macOS, where `cc`
is there.

The sources are looked for (a directory counts when it has the recipe's
first source, `sqlite3.c` or `bic_cxxrt.c`) in `$BIC_THIRD_PARTY`, then
`third_party/` beside the bic executable or one directory up (a package,
or bic built in a checkout) or in `share/bismuth/` one directory up,
beside `$BIC_STD`, in the program's directory and its parents (a
checkout, or a project that vendors them), and last where `bic install`
put the installed package's copy (`~/.local/share/bismuth/third_party`,
`/usr/local/share/bismuth/third_party` with `--system`,
`PREFIX/share/bismuth/third_party` with `--prefix`, and `third_party`
beside `bin` on Windows), which may be another version's than the bic
running. `bic_cxxrt.c`, bic's own part of the `openh264` recipe, is
compiled from the copy built into bic (written into the cache), never the
directory's, so an older install's copy is not linked in.

| Recipe | Sources | Flags |
|---|---|---|
| `sqlite3` | `third_party/sqlite/sqlite3.c`, the amalgamation (3.54.1, public domain) | `-O2 -DSQLITE_THREADSAFE=1 -DSQLITE_OMIT_LOAD_EXTENSION -DHAVE_USLEEP=1`, FTS5, R*Tree, math functions, column metadata |
| `openh264` | `third_party/openh264`: OpenH264 2.7.0's encoder (BSD), C++ without its assembly, and `bic_cxxrt.c` | `-O2 -fno-exceptions -fno-rtti -fno-strict-aliasing`; `bic_cxxrt.c` provides `operator new`/`delete` and the static-initialization guards, so no libstdc++ is linked |

Compiling the bundled C needs a C compiler on the build machine (not on
the machine that runs the program). SQLite's amalgamation takes about 25
seconds once. Where that is a problem, the alternative is to check in
prebuilt objects per target (`third_party/prebuilt/linux-x86_64/sqlite3.o`
and so on, made by CI) and have the recipe use them when no compiler is
found; that trades a reproducible source build for binaries in the
repository, and is not done.

### Status

Done:

- COFF objects and archives (and import libraries, for the DLLs) linked
  into Windows x86-64 executables, with the bundled SQLite and OpenH264:
  `tests/run/sqlite_bundled_windows.bi` and `h264_bundled_windows.bi` run
  on Windows (they need one of the compilers above there).
- ELF objects and archives linked into Linux x86-64 and arm64
  executables, and Mach-O objects and archives into macOS arm64 and x86-64
  ones: `@link(static: ...)`, `--link-static`, the recipes, and the
  cache. Tests (`tests/rt_cinterop.bi`, `tests/rt_misc.bi`): an archive
  built with `cc` (calls both ways, function pointers both ways, switch
  tables, absolute pointers in data, a common symbol, a weak reference, an
  unused member left out), the same with `--link-static` objects, the
  `-fPIC` error, the SQLite programs of `tests/run` with the bundled
  SQLite, and OpenH264 encoding frames. On macOS the same archive and
  objects (built with `cc` and `ar`, so a BSD archive with `__.SYMDEF`),
  `tests/run/sqlite_bundled_darwin.bi` and the OpenH264 test; the Mach-O
  path was checked structurally from Linux (every relocation of SQLite,
  OpenH264 and the test archive recomputed against the linked image, the
  bind and rebase tables, and the signature's page hashes) but has not yet
  run on a Mac.

Next:

- **Windows on Arm** (ARM64 COFF), with the native arm64 backend for
  Windows.
- **macOS unwind information**: merging the objects' `__compact_unwind`
  into an `__unwind_info` section (with Bismuth's own frames), so C++
  exceptions can cross C frames and crash reports unwind them.
- **Thread-local variables and static constructors**, for C++ libraries
  that need them (`.init_array` as `DT_INIT_ARRAY`, `PT_TLS`).
