# Bismuth standard library

The standard library is written in Bismuth (`std/`) on top of libc. There is
no C runtime: string building, formatting, hashing, panics, the thread pool
behind `parallel for` and tasks are ordinary Bismuth code, so the optimizer
sees and inlines all of it. On Windows "libc" is the Universal C Runtime
(`ucrtbase.dll`) plus kernel32 and Winsock; per-OS files (`*_linux.bi`,
`*_darwin.bi`, `*_windows.bi`, `*_unix.bi`) hold the parts that differ, so
every module below works the same on Linux, Windows and macOS.

Following the "library, not language" rule, only primitives (`bool`, the
integer and float types, `char`, `str`), structs, enums, tuples, fixed
arrays, pointers, nullables and results are native. Everything below is a
library type.

## Prelude (`std.core`, always visible)

| Area | Items |
|---|---|
| Results | `Result<T, E>` (`T ! E`): `isOk` `isErr` `or` `orElse` `expect` `unwrap` `ok` `err` |
| Errors | `RangeError` (from `T.check`), `DecodeError` (from `fromBytes`/`core.decode`), `ParseError` |
| Icons (SVG) | `loadIcon(svgText) : Icon ! SvgError`, `loadIconFile(path) : Icon ! SvgError`, `addIcon(Svg) : Icon`, `iconSvg(icon) : Svg`; `icon(icon, size, tint)`, `iconButton(label, icon, size, tint, selected) : bool` (drawn pressed while `selected`: a tool that is on), `drawIcon(icon, rect, tint)` (fitted into the rectangle, in pixels, on whole pixels). Sizes for `icon` and `iconButton` are at DPI scale 1 (times `scale()`), as font sizes are. Each icon is rasterized into the font atlas once per pixel size it is drawn at (again after `setScale`) and drawn from there texel for texel, in the same draw commands as text. With a `tint` an icon is a silhouette in that colour (its coverage, whatever its colours); without one it shows in its own colours, `currentColor` being the theme's `text`. `Svg.parse(text) : Svg ! SvgError` (`NotSvg`, `BadXml(offset)`, `Io(message)`), `svg.width`, `svg.height`, `svg.viewBox`, `svg.rasterize(w, h, currentColor) : image.Image` and `rasterizeRgba(w, h, currentColor) : List<u8>` (straight alpha; fitted and centred in w x h, aspect kept), `parsePath(d)`, `parseTransform(text) : SvgMatrix`, `parseColor(text) : Rgba8?`. Supported: `svg` (width, height, viewBox), `g`, `path` (every command, relative and absolute, implicit repeats, arcs), `rect` (rx, ry), `circle`, `ellipse`, `line`, `polyline`, `polygon`, `transform` on any element, `linearGradient` and `radialGradient` (gradientUnits, gradientTransform, stops, href; a radial gradient's focus is its centre), fill, stroke, fill-rule, fill-opacity, stroke-opacity, opacity (applied per shape, not to a group as one layer), color, currentColor, stroke-width, stroke-linecap, stroke-linejoin, stroke-miterlimit, display: none, as attributes or in `style`. Not supported: text, images, `use`, clip paths, masks, patterns, filters, markers, dashes, `<style>` sheets. Shapes are rasterized with exact area coverage (as glyphs are), curves flattened to within 1/20 pixel, the same pixels on every platform |
| Output | `print(str)`, `eprint(str)`, `printRaw(str)`; stdout is buffered (line-flushed on a terminal) |
| Panics | `panic(msg) : never`, `assert(cond, msg)`, `unreachable()`, `todo()` |
| Strings | `str`: `len` `isEmpty` `byteAt` `sub` `find` `rfind` `contains` `startsWith` `endsWith` `split` `lines` `trim` `trimStart` `trimEnd` `toUpper` `toLower` (Unicode's full case mappings, without locale or final-sigma context) `toAsciiUpper` `toAsciiLower` `replace` `repeat` `concat` `chars` `charCount` `bytes` `parseInt` (decimal, `0x` or `0b`; `Invalid` for values outside `int`, while hex and binary take any 64-bit pattern) `parseFloat` `cstr` |
| | `StrBuf`: `new` `withCapacity` `write` `writeByte` `writeChar` `writeBytes` `take` `str` `len` `clear` |
| | `char`: `isDigit` `isAlpha` `isAlnum` `isSpace` `isUpper` `isLower` `toAsciiUpper` `toAsciiLower` `code` (ASCII), and Unicode's `isLetter` `isNumber` `isAlphanumeric` `isWhitespace` and `toUpper` `toLower` (the simple case mapping: a character whose mapping is several, as ß's, stays itself) |
| Collections | `List<T>`: `new`/`List<T>()` `withCapacity` `len` `push` `pop` `get` `set` `first` `last` `insert` `remove` `swapRemove` `clear` `truncate` `capacity` `reserve` `shrinkToFit` `append` `drain(a, b)` (removes elements [a, b) and returns them) `reverse` `sort` `sortByInPlace` `stableSort` `stableSortByInPlace` `asSlice` `asMutSlice` (needs a list that may change), plus every `Slice` method. `sort` is an introsort, not stable; the `stable` ones are a merge sort that keeps elements comparing equal in their order |
| | `Slice<T>`: `len` `sub` `toList` `count` `countWhere` `filter` `map` `each` `fold` `find` `findIndex` `any` `all` `contains` `indexOf` `sum` `min` `max` `sortBy` (copy) `stableSortBy` (copy) `sorted` `reversed` `join`; read-only |
| | `MutSlice<T>`: a `Slice<T>` that writes its elements (`s[i] = v`, `for x in ref s`): `len` `sub` (a `MutSlice`) `asSlice` `reverseInPlace` `sortInPlace` `stableSortInPlace`, `atomicAdd(i, v)` (integer elements: adds atomically and returns the element before, as shader code's atomic counters do), plus every `Slice` method; converts to `Slice<T>` implicitly |
| | `Map<K, V>`: `new` `len` `set` `get` `has` `remove` `keys` `values` `slotOf` `clear`; `for k, v in map` |
| Memory | `Shared<T>` (what `new T { }` makes; fields reached through it; passes as a `view T`; `refCount`, `ptrEq(other)`: the same object, as `==` on two handles of one type), `Weak<T>` (`s.weak()`, `w.get()`, `ptrEq`), `Arena` (`Arena(4.mb)`, `alloc<T>(n)`, `reset`, `free`, `used`, `capacity`), `ArenaSlice<T>` (what `alloc` returns: `len`, indexing, `for`, `live`, `asSlice` and through it every `Slice` method, `asMutSlice`; each access panics with "arena slice used after reset/free" once the arena was reset or freed, except with `--unchecked`) |
| Heap | `BismuthAllocator` (`alloc` `allocZeroed` `realloc` `free` `usableSize`, figures `heapsInUse` `heapsMade` `dirtyBytes` `committedBytes` `segmentBytes` `hugeBytes` `hugeCachedBytes`; see [Allocators](#allocators)), `cMalloc` `cCalloc` `cRealloc` `cFree` (the C library's heap, whatever the program's allocator) |
| Numbers | blanket methods on every integer type: `abs` `min` `max` `clampTo` `sign` `isEven` `isOdd` `pow`, `leadingZeros` `trailingZeros` (an `int`: the zero bits above the highest one bit or below the lowest, the type's width for 0) `swapBytes` (the bytes reversed; `bsr`/`bsf` with a `cmov` and `bswap` on x86-64, `clz`, `rbit` and `rev` on arm64); `f32`/`f64`: `abs` `sqrt` `floor` `ceil` `round` `trunc` (toward zero, a float) `pow` `min` `max` `clampTo` `isNan` (+ `exp` `ln` `log2` `log10` on f64); generic `min` `max` `clamp` `abs` `sqrt` `lerp` `swap`; `PI` `TAU` `E` |
| Units | `90.deg` `1.57.rad` → `Angle`; `2.s` `16.ms` `5.us` `100.ns` `3.minutes` → `Duration`; `4.kb` `4.mb` `1.gb` → `int` |
| Angles | `Angle` with operators; `sin` `cos` `tan` take an `Angle`; `asin` `acos` `atan2` return one |
| Vectors | `vec2` `vec3` `vec4` `ivec2` `ivec3`: constructors (`vec3(1, 2, 3)`, `vec3(0)`, `vec4(v.zyx, 1.0)`), `+ - * /` with vectors and scalars, unary `-`, `dot` `cross` `length` `lengthSq` `normalized` `distance` `lerp` `abs` `min` `max`, all 2- and 3-component swizzles (`v.xz`, `v.zyx`) |
| Integer SIMD | `u32x4`: four u32 lanes in one SIMD register (SSE2 on x86-64, NEON on arm64): `new(x, y, z, w)` `splat(s)` `load(p)` `store(p)` (16 bytes, any alignment), `+ -` `and` `or` `xor`, `shl(n)` `shr(n)` `rotl(n)` (every lane, `n` mod 32), `zipLo` `zipHi` `lo64` `hi64` (the 4x4 transpose's interleaves); `u32x8`: eight lanes in one AVX2 ymm register, which baseline x86-64 need not have, so its code must only run when `u32x8.native()` (CPUID: the CPU has AVX2 and the OS saves ymm state, asked once per process and kept, so a call is a load and a test) says so (always on arm64, where it is two NEON registers): `u32x4`'s operations on 32 bytes, its shuffles within each 128-bit half, and `lo128` `hi128` joining halves; both also `*` (the low 32 bits of each product), `min` `max` `eq` `gt` `lt` (unsigned; a compare gives all ones in the lanes where it holds, else 0) and `blend(o, mask)` (each bit of `o` where `mask` has it, else `self`'s), and `u32x8` `lo()` `hi()` (a 128-bit half as a `u32x4`: `x.hi().store(p)` stores one) `withLo(h)` `withHi(h)` `u32x8.join(lo, hi)`. `u8x16`, `i16x8` and `i32x4` are the same 16 bytes as sixteen u8, eight i16 or four i32 lanes (no CPU check: SSE2, NEON on arm64), `u8x32`, `i16x16` and `i32x8` their 32-byte AVX2 forms (run only when `native()`, which is `u32x8.native()`); every one has `splat(s)` `fromArray(a)` `toArray()` `get(i)` `load(p)` `store(p)`, `+ -` (wrapping) `and` `or` `xor`, `eq` `gt` `lt` `min` `max` (signed but for `u8`) `blend`, and `asU8x16()` `asI16x8()` `asI32x4()` `asU32x4()` (or `asU8x32()` ... `asU32x8()`: the same bits as other lanes); the 32-byte ones `lo` `hi` `withLo` `withHi` `join` as `u32x8`. `u8x16`/`u8x32`: `addSat` `subSat` (clamped to 0..255), `sad(o)` (`psadbw`: the sums of the absolute differences of each eight bytes, in lanes 0 and 2 of a `u32x4`, the others 0), `zipLo` `zipHi` (byte interleaves), `widenLo()` `widenHi()` (bytes 0..7 or 8..15 zero-extended to an `i16x8`), `shuffle(idx)` (`pshufb`: lane i is `self[idx[i] & 15]`, or 0 where `idx[i]` >= 128; the 32-byte form looks up within each 128-bit half), which on 16 bytes is SSSE3 and must only run where `u8x16.ssse3()` (CPUID, cached like `native()`; true off x86-64, arm64's is `tbl`) says so. `i16x8`/`i16x16`: `new` (eight lanes), `addSat` `subSat` (clamped to -32768..32767), `*` (low 16 bits) `mulHi` (high 16 of the signed product), `madd(o)` (`pmaddwd`: lane i of the `i32x4` is `a[2i]*b[2i] + a[2i+1]*b[2i+1]`), `abs` (of -32768 is -32768), `shl(n)` `shr(n)` `sra(n)` (logical and arithmetic, `n` mod 16), `packU8(o)` (self's lanes then `o`'s, clamped to 0..255), `zipLo` `zipHi`, `widenLo()` `widenHi()` (sign-extended to an `i32x4`). `i16x8`/`i16x16` and `i32x4`/`i32x8`: `sign(o)` (`psignw`/`psignd`: self's lanes negated where `o`'s are negative, 0 where they are 0), `hadd(o)` (`phaddw`/`phaddd`: the wrapping sums of adjacent lanes, self's then `o`'s; also on `u32x4`/`u32x8`). `i32x4`/`i32x8`: `new`, `*` (low 32 bits), `abs`, `shl` `shr` `sra` (`n` mod 32), `pack(o)` (to an `i16x8`, clamped to -32768..32767), `zipLo` `zipHi`. All eight have `shrBytes(n)` `shlBytes(n)` (`psrldq`/`pslldq`: the 16 bytes moved down, toward byte 0, or up by `n`, 0..16, zeros shifted in; of the 32-byte ones each 128-bit half's; one instruction for a constant `n`, which `core.simdShrBytes`/`simdShlBytes` require), and the 16-byte ones `loadLo(p)` (8 bytes at `p` into the low half, the high half 0) and `storeLo(p)` (the low half's 8 bytes), `movq` on x86-64 and `ldr d`/`str d` on arm64. The 32-byte forms' zips, packs, `sad`, `madd` and byte shifts work within each 128-bit half, as AVX2's do. On x86-64 each operation is one SSE2 (16 bytes) or AVX2 (32 bytes) instruction, but for the unsigned compares (a signed compare of lanes biased by the sign bit) and, on 16 bytes, `*`, `min` and `max` of 32-bit lanes, `abs`, `sign` and `hadd`, which need SSE4.1 or SSSE3 and are short SSE2 sequences instead. The 16-byte types need no CPU check but for `u8x16.shuffle`; On arm64 they are NEON: one instruction each but for `mulHi` and `madd` (`smull`, `smull2` and `uzp2` or `addp`), `sad` (`uabd` and three `uaddlp`), the packs (two narrowing moves), `sign` (five), `shuffle` (`tbl` of the indexes masked with `0x8f`) and the byte shifts (`ext` with zeros); the 32-byte types are two registers. `core.simdNative()` (a compile-time constant) is true where they are SIMD instructions (x86-64, arm64) and false where they would run as scalars, for code that keeps a scalar version for the latter. `u64x2` (16 bytes, no CPU check) and `u64x4` (32 bytes, AVX2: only when `u64x4.native()`) hold two or four u64 lanes: `new` `splat` `fromArray` `toArray` `get` `load` `store`, `+ -` (wrapping), `mulLo32(o)` (`pmuludq`: each lane's full 64-bit product of the low 32 bits of `self`'s and `o`'s lanes), `shl(n)` `shr(n)` (`n` mod 64), `and` `or` `xor`, `zipLo` `zipHi` (`punpcklqdq`, `punpckhqdq`; of `u64x4` within each 128-bit half, and `lo128` `hi128` joining halves), `shrBytes` `shlBytes`, `asU8x16()` `asU32x4()` (`asU8x32()` `asU32x8()`; `u32x4.asU64x2()` and `u32x8.asU64x4()` back), and `u64x4` `lo` `hi` `withLo` `withHi` `join` as `u32x8`: the operations of multi-limb arithmetic in 26- or 32-bit limbs, as Poly1305's AVX2 code (`paddq` `psubq` `psllq` `psrlq` `pmuludq` on x86-64, `add` `sub` `shl` `ushr` `.2d` and `xtn` + `umull` on arm64) |
| Float SIMD | `f32x4`: four f32 lanes in one SIMD register (SSE on x86-64, NEON on arm64), `f64x2` two f64 lanes: `new` `splat(s)` `zero()` `load(p)` `store(p)` (16 bytes, any alignment), `+ - * /` (with another vector, or `*` and `/` by a scalar), unary `-`, `min(o)` `max(o)` (the scalar `min`/`max` lane by lane: where `o`'s lane is NaN, self's), `sqrt()` `abs()`, `sum()` (`((x + y) + z) + w`) and `dot(o)`; `f32x4.fromVec4(v)` and `toVec4()`. Every lane is the scalar IEEE operation, never fused, so a loop rewritten with them gives the scalar loop's bits (a 4x4 matrix product in glm's column order is `a0 * splat(b.x) + a1 * splat(b.y) + a2 * splat(b.z) + a3 * splat(b.w)` per column). The methods are the `core.simdF*` intrinsics (`simdFAdd` `simdFSub` `simdFMul` `simdFDiv` `simdFMin` `simdFMax` `simdFSqrt` `simdFAbs` `simdFNeg` `simdFSplat<T>`), and `core.simdLoad`/`simdStore` take them too |
| Matrices | `mat4`: `identity` `translate` `scale` `rotateX/Y/Z` `perspective`, `m * m`, `m * vec4`, `transposed`; `quat`: `identity` `axisAngle`, `q * q`, `q * vec3`, `conjugate` |
| Concurrency | `Task<T>` (`spawn`, `await`, `done`, `cancel`), `all` `join` `race` `select` `timeout`, `sleep(d)`, `yieldNow()`, `blocking(f)`, `Atomic<T>` (`load` `store` `add` `sub` `swap` `compareSwap`), `Mutex<T>` (`lock(fn(ref T))` `tryLock` `get` `set`; parks a waiting task, may be held across `await`), `Chan<T>` (`new(cap)` `unbounded()` `send` `sendTimeout` `trySend` `recv` `recvTimeout` `tryRecv` `close` `isClosed` `sender()`, `for x in ch`), `Sender<T>` (closes its channel when the last one goes), `catchPanics()` (a panic then ends only the calling task: `Task.wait()`, `Task.panicked() : str?`; awaiting a panicked task with a result panics the awaiter), `Task.detach()`. Tasks run M:N on a few threads; see [Concurrency](../wiki/Concurrency.md) |

## Namespaces (`import std.<name>;`)

Every module outside the prelude lives in a namespace under `std`, written
with dots as in C#: `import std.net.http;`, then `http.get(url)`. The last
part of the name is what the code uses, or `as` picks another
(`import std.net.http as web;`). A namespace that only groups others
(`std.io`, `std.interop`) has nothing to import itself.

| Namespace | Contents |
|---|---|
| `std.os` | `args()`, `env(name) : str?`, `setEnv(name, value)`, `unsetEnv(name)`, `envVars() : List<(str, str)>`, `exePath() : str ! fs.IoError` (this program's executable, absolute: `/proc/self/exe`, `_NSGetExecutablePath` resolved with realpath, `GetModuleFileNameW`), `currentDir() : str ! fs.IoError` (the working directory: getcwd, `GetCurrentDirectoryW`), `setCurrentDir(path) : void ! fs.IoError` (chdir, `SetCurrentDirectoryW`; for the whole process), `exit(code)`, `pid()`, `hostname()`, `cpuCount()`, `setWorkers(n)` (worker threads for tasks and `parallel for`, before the first use; also `BISMUTH_WORKERS`), `schedStats() : SchedStats` (the scheduler's workers, fibers and stack memory), `deepestStacks(n) : List<StackDepth>` (the fibers whose stacks went deepest, with the spawn site), `memStats() : MemStats` (resident memory, its peak and the C heap; see [Allocators](#allocators)); the target as constants: `OS` (`"linux"`, `"windows"` or `"macos"`) and `ARCH` (`"x86_64"` or `"aarch64"`), the names Rust's `std::env::consts` and `uname` use, for what bic's `--target` and `--arch` chose, and `IS_WINDOWS` `IS_UNIX` |
| `std.os.process` | `run(argv, opts) : Output ! ProcessError` (exit status, stdout, stderr), `start(argv, opts) : Child`; `Options { stdin, stdout, stderr, cwd, clearEnv, env, creationFlags, newSession, newProcessGroup }` with `setEnv` `removeEnv`, `Stdio` `.inherit` `.pipe` `.discard`; `Child.pid` `writeStdin` `writeStdinBytes` `closeStdin` `closeStdout` `closeStderr` `readStdout` `readStderr` `pollStdout(ms)` `pollStderr(ms)` `readStdoutText` `readStderrText` `output(input)` `wait` `tryWait` `kill` `terminate`; `ExitStatus { code, signal }` with `success()`. See [Processes](#processes) |
| `std.os.pty` | pseudo-terminals, for running shells: `start(argv, opts) : Pty ! PtyError`; `Options { size, cwd, clearEnv, env }` with `setEnv` `removeEnv`, `Size { cols, rows }`; `Pty.read(ref buf, max) : int` (0 once the terminal is closed; parks a task) `readAll` `write` `writeBytes` `resize(cols, rows)` `size` `pid` `kill` `wait` `tryWait` (`process.ExitStatus`) `close` `clone`; `PtyError` (`Unsupported` `NotFound` `PermissionDenied` `InvalidInput` `Closed` `Cancelled` `Io`). posix_openpt on Linux and macOS, ConPTY on Windows. See [Pseudo-terminals](#pseudo-terminals) |
| `std.os.dbus` | D-Bus clients without libdbus: `session(opts)`, `system(opts)`, `connect(address, Options { hello, fds, timeout }) : Conn ! DbusError` (async); `Conn.call(msg, timeout) : Message` `callMethod(dest, path, iface, member, args) : List<Value>` `send` `waitReply` `receive(timeout)` `waitSignal(rule, timeout)` `subscribe(rule) : Signals` (`for m in ...`) `addMatch` `removeMatch` `emit` `reply` `replyError` `requestName` `releaseName` `nameOwner` `listNames` `getProperty` `setProperty` `getAllProperties` `uniqueName` `canPassFds` `close`; `Message` (`methodCall`, `signal`, `methodReturn`, `errorReply`; `encode`, `decodeMessage`), `Value` (every D-Bus type; `sig` `str` `asStr` `asU32` `asInt` ... `items` `entries` `get` `inner` `equals`), `variant` `array` `strs` `dict` `VarDict`, `Match` (`rule`, `matches`), `marshal` `unmarshal` `validSignature` `parseAddresses`; Linux; see [below](#d-bus) |
| `std.os.unix` | Linux and macOS: `ioctl` (`unsafe`), `ioctlInt`, `ioctlWith<T>` (size-checked), `io` `ior` `iow` `iowr` `ioc` `iocDir` `iocSize`; TUN devices `openTun(TunOptions { name, kind, packetInfo, nonBlocking, multiQueue }) : Tun` (`read` `write` `fd` `name` `mtu` `setMtu` `up` `down` `setAddress` `setNonBlocking` `close`; Linux `setPersist` `setOwner` `setGroup`), `interfaceFlags` `interfaceMtu` `setInterfaceMtu` `setInterfaceUp` `setInterfaceAddress`; file watching `watcher() : Watcher` (`add` `remove` `read(timeoutMs) : List<Change>` `fd` `paths` `close`), Linux `inotifyInit() : Inotify` (`addWatch` `rmWatch` `read`), `parseInotifyEvents`; `pid` `ppid` `setsid` `getsid` `setpgid` `getpgid` `kill` `killpg` `processExists`; `uid` `euid` `gid` `egid` `setuid` `setgid` `userById` `userByName` `userName`; `getrlimit` `setrlimit` (`Resource`, `Limit { soft, hard }`) `raiseNofileLimit()`; `uname()`, `statvfs(path) : FsStats`, `sysinfo() : SysInfo`, `loadavg`, `uptime`; `setNonBlocking` `setCloseOnExec` `waitReadable`; `parseIpv4` `prefixMask`; errors are `SysError { call, code }`. See [below](#unix) |
| `std.os.windows` | Windows: registry `openKey(Root, path, Access) : Key`, `createKey`, `openCurrentUser`, `getValue(Root, path, name, expand)`, `keyOf`, `rootKey`, `deleteKey` `deleteTree` `keyExists`, `Key.get : Value` `getValue(sub, name, expand)` `getStr` `getDword` `getQword` `getBinary` `getMultiStr` `set` `setStr` `setExpandStr` `setDword` `setQword` `setBinary` `setMultiStr` `has` `subkeys` `values` `deleteValue` `deleteSubKey` `deleteSubTree` `open` `create` `close`; jobs `createJob() : Job` (`assign` `assignPid` `setLimits(JobLimits { killOnClose, breakawayOk, silentBreakawayOk, ... })` `setKillOnClose` `activeProcesses` `processIds` `contains` `terminate` `close`), `inAnyJob`; `openProcess(pid) : Process` (`terminate` `wait` `exitCode` `isRunning` `path` `token` `close`), `terminatePid`, `currentPid`, `listProcesses() : List<ProcessEntry>`; `startProcess(commandLine, StartOptions { cwd, env, desktop, flags, parent, jobs, inherit, stdIn, stdOut, stdErr, show }) : Started`, `startAsUser(token, ...)`, `resumeThread`, `makeInheritable`, `CREATE_*`; tokens `processToken` `threadToken` `sessionUserToken` `tokenOf`, `Token` (`user` `isSystem` `sessionId` `setSessionId` `isElevated` `integrityLevel` `linked` `duplicate` `enablePrivilege` `impersonate` `environment` `profileDir` `info` `close`), `revertToSelf`, `enablePrivilege`, `SE_*_NAME`; `Sid` (`str` `bytes` `ptr` `authority` `subauthorities` `rid` `isWellKnown` `account`), `parseSid`; sessions `activeConsoleSession` `processSession` `sessions() : List<WtsSession>` `sessionUser`; desktops `openInputDesktop` `openDesktop` `threadDesktop` (`name` `setThread` `close`); `sendInput(List<Input>)`, `mouseMove` `mouseMoveTo` `mouseWheel` `keyEvent` `unicodeKey`; `windowPlacement` `setWindowPlacement`; `setProcessDpiAwareness` `setThreadDpiAwareness` `restoreThreadDpiAwareness` `dpiForWindow` `dpiForSystem`; CCD `queryDisplayConfig() : DisplayConfig { paths, modes }` `setDisplayConfig` `sourceName` `targetName`; `deviceInterfaces(classGuid)`; `knownFolder(KnownFolder)` `knownFolderById`, `recycleBin` `emptyRecycleBin` `recycle(paths)`; `thread(ctx, fn) : OsThread` (`join` `isRunning` `close`), `mmcss(task) : Mmcss` (`setPriority` `revert`), `setProcessPowerThrottling` `setThreadPowerThrottling`, `SrwLock` (`lock` `unlock` `lockShared` `unlockShared` `tryLock` `tryLockShared` `with`); `cpuTimes` `cpuUsage` `memoryStatus` `diskSpace` `uptimeMs`; `adapters() : List<Adapter>`; `loadWintun() : Wintun` (`createAdapter` `openAdapter` `driverVersion` `deleteDriver` `logToStderr`), `WintunAdapter` (`luid` `startSession` `close`), `WintunSession` (`receive` `read` `send` `readWaitEvent` `end`), `parseGuid`; `toUtf16` `wide` `fromUtf16` `fromWide` `fromWideZ` `wideLen` `ipText` (these work on every OS); `WinError { call, code }` (`message` `isNotFound` `isAccessDenied`), `lastError`, `errorMessage`. See [below](#windows) |
| `std.os.windows.service` | Windows services: `run(name, fn(Service), RunOptions { accept..., startWaitHint, stopWaitHint, reportStopPending }) : void ! WinError` (`Service.controls() : Chan<Control>` `args` `setRunning` `setPaused` `setStartPending` `setStopPending` `setExitCode` `setServiceExitCode` `state`); through the SCM `install(Config)` `uninstall` `exists` `start(name, args)` `stop` `pause` `resume` `sendControl` `query(name) : Status` `waitFor(name, State, timeoutMs)` `exePath`. See [below](#windows-services) |
| `std.os.windows.dxgi` | DXGI, Windows only (dxgi.dll loaded at run time): `createFactory1() : Factory ! com.HrError`, `createFactory2(flags)`; `Factory` (`enumAdapters1(i)` `adapters()` `isCurrent` `makeWindowAssociation` `createSwapChain` `createSwapChainForHwnd` `enumWarpAdapter` `enumAdapterByLuid`), `Adapter` (`enumOutputs(i)` `outputs()` `getDesc` `getDesc1`), `Output` (`getDesc` `getDisplayModeList` `waitForVBlank` `duplicateOutput(device)` `duplicateOutput1(device, formats)`), `OutputDuplication` (`getDesc` `acquireNextFrame(ms) : AcquiredFrame { info, resource }` `getFrameDirtyRects` `getFrameMoveRects` `getFramePointerShape` `mapDesktopSurface` `unMapDesktopSurface` `releaseFrame`), `Device` (`getAdapter` `setGPUThreadPriority` `getGPUThreadPriority`), `Resource` (`getDevice` `getSharedHandle` `getUsage`), `Surface` (`getDesc` `map` `unmap`), `SwapChain` (`present` `getBuffer` `getDesc` `resizeBuffers` `getContainingOutput`), all extending `Object` (`ptr`, a `com.Ptr`; `raw` `cast` `getParent`); layouts `AdapterDesc` `AdapterDesc1` (`name()` `isSoftware()`) `OutputDesc` `ModeDesc` `SampleDesc` `Rational` `SwapChainDesc` `SwapChainDesc1` `SurfaceDesc` `MappedRect` `OutduplDesc` `OutduplFrameInfo` `OutduplMoveRect` `OutduplPointerShapeInfo` `Point`, `Rect` and `Luid` (std.os.windows's); `FORMAT_*` `COLOR_SPACE_*` `USAGE_*` `SWAP_EFFECT_*` `PRESENT_*` `ERROR_*` (`errorName(hr)`), `IID_IDXGI*`; each vtable slot as `IDXGI<INTERFACE>_<METHOD>` with its function type. See [below](#direct3d-11-and-dxgi) |
| `std.os.windows.d3d11` | Direct3D 11, Windows only (d3d11.dll loaded at run time): `createDevice(driverType, flags, featureLevels, adapter) : Created { device, context, featureLevel } ! com.HrError`, `createDeviceAndSwapChain`; `Device` (`createBuffer` `createTexture2D` `createShaderResourceView` `createUnorderedAccessView` `createRenderTargetView` `createDepthStencilView` `createInputLayout` `createVertexShader` `createPixelShader` `createComputeShader` `createBlendState` `createDepthStencilState` `createRasterizerState` `createSamplerState` `createQuery` `createDeferredContext` `openSharedResource` `checkFormatSupport` `checkMultisampleQualityLevels` `getFeatureLevel` `getCreationFlags` `getDeviceRemovedReason` `getImmediateContext` `dxgiDevice` `multithread` `videoDevice`), `DeviceContext` (`map` `unmap` `copyResource` `copySubresourceRegion` `updateSubresource` `resolveSubresource` `generateMips` the clears, the `ia*` `vs*` `ps*` `cs*` `om*` `rs*` setters, `draw*` `dispatch` `begin` `end` `getData` `clearState` `flush` `getType` `finishCommandList` `executeCommandList` `context1` `videoContext`), `DeviceContext1` (`copySubresourceRegion1` `discardResource` `discardView` `vsSetConstantBuffers1` `psSetConstantBuffers1` `csSetConstantBuffers1` `clearView`), `Resource` `Buffer` `Texture2D` (`getDesc`), the views, shaders and state objects, `Query`, `CommandList`, `Multithread`, `VideoDevice` `VideoContext` `VideoContext1` `VideoProcessorEnumerator` `VideoProcessor` and its views; layouts `BufferDesc` `Texture2DDesc` `SubresourceData` `MappedSubresource` `Box` `Viewport` `Rect` `BlendDesc` `RenderTargetBlendDesc` `RasterizerDesc` `DepthStencilDesc` `DepthStencilOpDesc` `SamplerDesc` `InputElementDesc` `QueryDesc` `QueryDataTimestampDisjoint` the view descs (`buffer` `texture2D`) and the video processor's; `DRIVER_TYPE_*` `FEATURE_LEVEL_*` `CREATE_DEVICE_*` `USAGE_*` `BIND_*` `CPU_ACCESS_*` `MAP_*` and the rest, `IID_ID3D11*`; each vtable slot as `ID3D11<INTERFACE>_<METHOD>` with its function type. See [below](#direct3d-11-and-dxgi) |
| `std.os.signal` | signals as values: `listen(sigs) : Chan<int>` (each signal as it arrives; listening replaces the default action until `reset(sig)`), `ctrlC()` and `shutdown()` (async: the next Ctrl-C; the next TERM, INT or HUP), `name(sig)`; `INT` `TERM` `HUP` `QUIT` `USR1` `USR2` `WINCH` `CHLD` `ALRM`. On Windows: Ctrl-C (`INT`), Ctrl-Break (`QUIT`), console close (`HUP`), logoff and shutdown (`TERM`) |
| `std.sync` | coordination between tasks (Tokio's sync): `Oneshot<T>` (`send` once, `recv` `recvTimeout` `tryRecv` `close`), `Notify` (`notifyOne` `notifyAll` `wait` `waitTimeout`), `Semaphore` (`acquire(n)` `acquireTimeout` `tryAcquire` `release(n)` `with(f)` `available` `close`), `RwLock<T>` (`read(f)` `write(f)` `get` `set`, writers first), `Watch<T>` (`send` `update` `get` `watch() : Watcher<T>` with `changed` `changedTimeout` `get` `hasChanged`), `Broadcast<T>` (`send` `subscribe() : Subscriber<T>` with `recv` `recvTimeout` `tryRecv` `lagged`, `for v in sub`), `CancelToken` (`cancel` `isCancelled` `cancelled()` `cancelledTimeout` `child()`), `WaitGroup` (`add` `done` `wait`), `Barrier` (`wait`) |
| `std.stream` | streams of values and bytes (tokio-util, futures-util): any iterator is a stream (`for`, `linq.fromIter`); `merge(sources) : Chan<T>`, `forward(it, sender)`, `buffered(it, n, f)` and `bufferUnordered(it, n, f)` (up to n calls of f at once, in detached tasks, so f must not capture the caller's locals); codecs over a `net.Stream`: `lengthDelimited() : LengthCodec { prefixBytes, bigEndian, maxFrame }` with `read` `write` `frames(s)`, `lines(s, maxLen)`, `chunks(s, size)`; `chunkReader(it)` reads a stream of byte lists as bytes (`read` `readExact` `readLine` `readAll`) |
| `std.io.poll` | readiness of any descriptor (mio): `readable(fd, timeout)` and `writable(fd, timeout)` (async; in a task through the scheduler's poller), `forget(fd)` before closing one, `setNonBlocking(fd)`, `wouldBlock()`, `waitHandle(h, timeout)` (Windows: events, processes and other waitable handles); `PollError` (`Timeout` `Cancelled` `Closed` `Failed`) |
| `std.io.fs` | `read(path) : List<u8> ! IoError`, `readText`, `write`, `append`, `writeBytes`, `exists`, `remove`, `setExecutable`, `makeDir`, `makeDirAll`, `removeDir`, `removeAll`, `listDir`, `stat` and `lstat` (`FileInfo`: kind, size, mode, times, inode, device, links, uid, gid), `rename`, `realPath`, `symlink`, `readLink`, `setTimes`, `chmod`, `chown`, `syncDir`, `open(path, mode, perm) : File` (modes `read` `write` `append` `readWrite` `createNew`) with `read` `readAll` `write` (text) `writeBytes` (a `List<u8>`) `writeSlice` (a `Slice<u8>`) `sync` `close`; wide (UTF-16) paths on Windows |
| `std.io.stdin` | standard input: `readLine() : str?` (no line ending; null at the end of input), `prompt(question) : str?`, `lines()` (`for line in stdin.lines()`), `readAll() : str`; one shared buffer, so they mix without losing input |
| `std.time` | `now() : Instant` (`elapsed`, `-`), `unixMillis()`, `sleep(d)`; calendar `Date`, `Time`, `DateTime`, wall-clock `Timestamp`, `Zone` (UTC, fixed offsets, IANA TZif files, POSIX TZ rules, `localZone()`), `ZonedDateTime`, RFC 3339 and `strftime`; see [below](#time) |
| `std.cli` | command-line flags: `Command.new(name, about)` with `boolFlag` `intFlag` `floatFlag` `strFlag` `listFlag` `arg` `restArgs` `command`; `parse(args) : Matches ! CliError`, `parseOrExit`, `parseOs`, `help`, `usage`; see [below](#cli) |
| `std.rand` | `Rng.new(seed)` / `Rng.fromTime()` with `next` `next32` `below(n)` `range(lo, hi)` `index(n)` (unbiased, Lemire's method) `float` `float32` `uniform` `chance` `bernoulli` `normal` `exponential` `poisson` `shuffle` `choose` `inCircle` `onCircle` `inSphere` `onSphere` `inDisc`; `Weighted.new(weights) : Weighted ! WeightError` with `sample(ref rng)` (alias method); global `rand.range(lo, hi)`, `rand.float()`, `rand.seed(s)`; see [below](#rand) |
| `std.noise` | `Noise.new(seed)` with `perlin` `simplex` (OpenSimplex2) `simplexXY` `value` `cellular(.., Metric, jitter) : Cell` (`f1` `f2` `id` `value()`) `fbm` `ridged` `turbulence` (`Fractal { basis, octaves, lacunarity, gain }`), in 2D and 3D; see [below](#noise) |
| `std.collision` | `Aabb2` `Aabb3` `Circle` `Sphere` `Capsule2` `Capsule3` `Segment2` `Segment3` `Triangle` `Plane` with `overlaps` `contains` `closestPoint`; `Ray2`/`Ray3.cast(shape, maxT) : Hit?`; `Segment2.intersect`, `closestPoints`; `pointInPolygon`, `sat(a, b) : Contact2?`, `closestPointOnPolygon`; `Grid<B>` and `AabbTree<B>` broadphases; see [below](#collision) |
| `std.net` | Messages: `listen(addr, opts) : Server`, `connect(addr, opts) : Conn` (async), `Conn.send/recv/request` (async), `Server.messages<T>()`, `receive<T>`, `broadcast`, `accept`, `port`; `ListenOptions { proto, reliable, compress, maxClients, timeoutMs, ... }`. Byte streams: `tcpConnect(addr, timeout) : Stream`, `tcpListen(addr) : Listener` (`accept(timeout)`, `port`, `localAddr`, `close`), Unix domain sockets `unixConnect(path) : Stream` and `unixListen(path)`/`unixListenFresh(path) : Listener` (`@name`: Linux's abstract namespace), `Stream.read` `readExact` `readExactText` `readLine` `readAll` `readText` `write` `writeAll` `writeText` `writeRaw` `waitReadable` `setReadTimeout` `setWriteTimeout` `setTimeout` `setNoDelay` `setKeepAlive` `shutdown` `localAddr` `peerAddr` `isTls` `tlsVersion` `close`. Datagrams: `udpBind(addr) : UdpSocket` and `udpConnect(addr) : UdpSocket` (connected to one peer) (`sendTo(bytes, Addr)`, `connect(Addr)`, `send(bytes) : int` and `recv(ref buf, timeout) : int` (connected: to and from the peer only, `Refused` once its port is closed), `peerAddr`, `recvFrom(timeout) : Datagram`, `recvInto(ref buf, timeout) : Addr` (into a reused list: no allocation a datagram), `recvBatch(ref Datagrams, timeout) : int`, `sendBatch(Datagrams) : int`, `trySendBatch(Datagrams, from) : int` (without waiting), `setGro(on) : bool` (UDP GRO: recvBatch takes coalesced runs, Linux), `setBroadcast`, `port`); `Datagrams` (`new(count, size)` `len` `capacity` `size` `clear` `bytes(i)` `from(i)` `push(bytes, Addr)` `slot` `commit(n, Addr)` `ptr(i)` `setLen(i, n)`: reusable buffers, one system call a batch: see [Networking](#networking)). Socket options: `SockOptions { reuseAddr, reusePort, recvBuffer, sendBuffer, v6Only, noDelay, keepAlive, keepAliveParams, linger, device, bindAddr, ttl }` for `tcpListen(addr, backlog, opts)`, `tcpConnect(addr, timeout, opts)`, `udpBind(addr, opts)`; `KeepAlive { idle, interval, count }`; on open sockets `noDelay` `keepAlive` `setKeepAliveParams` `keepAliveParams` `setLinger` `linger` `setRecvBufferSize` `recvBufferSize` `setSendBufferSize` `sendBufferSize` `setTtl` `ttl` `reuseAddr` `reusePort` `v6Only` `bindDevice` `setRawOption` `rawOption` (see [Networking](#networking)). Names: `lookup(host) : List<str>`, `resolveAddr(addr) : Addr` (`ip` `port` `isV6` `str`, IPv6 as RFC 5952 text) |
| `std.net.tls` (also `net.tls` after `import std.net;`) | TLS 1.3 and 1.2 written in Bismuth (no OpenSSL): `connect(addr, Options { verify, caFile, caDir, roots, systemRoots, verifier, serverName, alpn, certFile, keyFile, clientCert, minVersion, maxVersion, cipherSuites, groups, timeout }) : net.Stream` (async), `client(stream, host, opts)` (TLS on a connected stream); `serverConfig(certFile, keyFile)`, `serverConfigPem`, `ServerConfig.new(ServerOptions { certs, resolve, alpn, clientAuth, clientRoots, ... })` with `accept(stream)`, `update`, `setCertificate`; `CertifiedKey`, `Verifier` (`standard` `pinned` `custom` `none`), `info(stream)` (version, suite, ALPN, SNI, peer certificates), `keyUpdate(stream)`; see [TLS](#tls) |
| `std.net.http` (also `net.http` after `import std.net;`) | Client: `get(url) : Response ! HttpError`, `head`, `delete`, `post(url, body, contentType)`, `put`, `patch`, `postJson`, `getJson<T>`, `send(Request { method, url, headers, body, bodyReader, upload, timeout, connectTimeout, totalTimeout, maxRedirects, allowRedirect, maxResponseBytes, tlsOptions, proxy, target })`, `open(req)` (the body streamed: `resp.reader()`), `upgrade(req) : Upgraded` (101 and CONNECT); `Client.new()` (keep-alive) with the same methods; `producer(fn, length)` for uploads; http and https. Server: `Server.new()`, `route(method, pattern, handler)`, `get` `post` `put` `delete` `patch` `options` `any` `connect` `stream` (body read by the handler) `routeWith(method, pattern, RouteOptions, handler)` `websocket` `websocketWith` `static(prefix, dir)` `use(middleware)` `nest(prefix, server)` `fallback`, `handle`, `listen(addr)`, `serve(listener)`, `serveTls(listener, cfg)`, limits `maxBodyBytes` `maxHeaderBytes` `maxHeaders` `maxConnections` `idleTimeout` `readTimeout` (0 for `maxBodyBytes`, `maxConnections` and the timeouts: no limit), `onPanic` `panicResponse`; `serve(addr, handler)`. `Request` (`path` (decoded) `rawPath` `header` `param` `queryParam` `queryPairs` `queryAs<T>` `form` `formAs<T>` `readJson<T>` `bodyReader` `reader` `stream(head, fn)` `sse(fn)` `upgrade(head, fn)` `tunnel(fn)` `canUpgrade`), `Response` (`status` `reason` `body` `bodyReader` `fields` `header` `reader` `readJson<T>`; `Response.text/html/json/jsonOf/new/redirect`, `withHeader`, `withReader`, `gzipFor`), `statusText(code)`, `Next` (`run`), `BodyReader` (`ofText` `ofBytes` `ofFile` `ofStream`; `read` `chunk` `chunks` `readLine` `readAll` `readText` `saveTo` `discard` `close` `isDone` `error` `contentLength` `sendLength`), `BodyWriter` (`write` `writeText` `flush` `finish` `abort`), `bodyPipe(length, capacity) : (BodyWriter, BodyReader)`, `SseWriter` (`send(Event)` `data` `event` `comment`), `SseReader` (`next`, `for await`), `Event`, `pipe(a, b)`, `serveFile`, `serveDir`, `decodeForm<T>`, `FormError`, `Headers`, `Url` (`parseUrl`, `resolve`, `hostPort`, `target`), `urlEncode`, `urlDecode`, `parseQuery`, `formEncode`, `basicAuth`, `matchPath` |
| `std.net.ws` | WebSocket (RFC 6455, permessage-deflate from RFC 7692): `connect(url, Options { protocols, headers, timeout, tlsOptions, proxy, maxMessage, maxFrame, fragmentSize, compress, compressLevel, onPong }) : Socket` (ws:// and wss://), `connectOn(stream, url, opts)` (the handshake over a stream already connected), `Socket.send(text)` `sendBinary(bytes)` `sendFragmented` `sendBinaryFragmented` `recv() : Message` `ping` `pong` `lastPong` `close(code, reason)` `protocol` `extensions` `isCompressed`; servers through `http.Server.websocket` / `websocketWith(pattern, ServerOptions { protocols, maxMessage, maxFrame, fragmentSize, compress })`, or `negotiate(req, opts)` and `accept(stream, handshake, opts)` in a handler of your own (`req.canUpgrade()`) |
| `std.net.ssh` | SSH-2 client and server: `connect(addr, Config) : Client` (async) with `authPublicKey` `authPassword` `authKeyboardInteractive` `authAgent` `authNone`, `exec(cmd) : ExecOutput`, `openSession() : Channel` (`requestPty` `exec` `shell` `subsystem` `setEnv` `resize` `signal` `read` `readStderr` `readAny` `write` `eof` `wait` `close`), `openDirectTcpip`, `forwardLocal`, `requestRemoteForward` `acceptForwarded` `serveRemoteForward`; `ServerConfig` with `onPassword` `onPublicKey` `onKeyboardInteractive` `onSession` `onDirectTcpip` `onTcpipForward`, `serve(listener, cfg)`, `serveConn(stream, cfg)`; `PublicKey`, `PrivateKey` (OpenSSH and PEM files, passphrases), known_hosts and authorized_keys, `Agent`; see [below](#ssh) |
| `std.net.ssh.sftp` | SFTP version 3: `open(client) : Sftp` with `read` `write` `openFile` `stat` `lstat` `setstat` `readDir` `mkdir` `rmdir` `remove` `rename` `posixRename` `realpath` `readlink` `symlink`, `File` (`read` `readAt` `write` `writeAt` `readAll` `stat` `close`), `Attrs`; `serve(channel, root, ServeOptions)` for servers; see [below](#sftp) |
| `std.net.quic` | QUIC v1 (RFC 9000, 9001, 9002) with datagrams (RFC 9221), in Bismuth on `std.net.tls`: `connect(addr, ClientConfig { tls, transport }) : Connection`, `listen(addr, ServerConfig { tls, transport, retry, maxConnections }) : Endpoint`, `endpoint(addr)`; `Endpoint.connect` `accept(timeout)` `port` `localAddr` `connectionCount`; `Connection.openStream` `openUniStream` `acceptStream` `acceptUniStream` `sendDatagram` `recvDatagram` `maxDatagramSize` `close(code, reason)` `closed` `isClosed` `remoteAddr` `alpn` `info` `stats` `keyUpdate`; `Stream.read` `readAll` `readExact` `readText` `write` `writeAll` `writeText` `finish` `flush` `reset(code)` `stopSending(code)` `setReadTimeout` `setWriteTimeout`; `Options { idleTimeout, keepAlive, handshakeTimeout, maxStreamsBidi, maxStreamsUni, streamWindow, connectionWindow, sendBuffer, maxDatagramSize, datagramQueue, mtuDiscovery }`, `QuicError`; see [QUIC](#quic) |
| `std.net.http3` | HTTP/3 (RFC 9114) with QPACK (RFC 9204) and WebTransport: client `connect(addr, ClientConfig { tls, transport, http })` → `Client` (`send` `get` `post` `open` → `Exchange` with `write` `writeText` `finish` `response` `read` `readAll` `cancel`; `webtransport(path)`), server `Server.new(fn(ref ServerRequest))` / `Server.routes(http.Server)` with `webtransport(pattern, fn(Session))`, `listen(addr, ServerConfig)`, `serve(endpoint)`; `ServerRequest` (`head`, `read` `readAll` `respond` `write` `finish` `send`); `Session` (`acceptStream` `acceptUniStream` `openStream` `openUniStream` `sendDatagram` `recvDatagram` `close`); `Options { qpackTableCapacity, maxFieldSectionSize, webtransport, maxBodyBytes }`, `H3Error`; QPACK's `encodeFields`, `Decoder`, `huffmanEncode`/`huffmanDecode`; see [HTTP/3](#http3-and-webtransport) |
| `std.net.url` | WHATWG URLs: `parse(text) : Url ! UrlError`, `parseWith(text, base)`, `resolve(base, text)`, `canParse`, `normalize`; `Url` getters `href` `protocol` `scheme` `username` `password` `host` `hostname` `port` `portOrDefault` `pathname` `segments` `search` `query` `hash` `fragment` `origin` `hostKind`, setters `setHref` `setProtocol` `setUsername` `setPassword` `setHost` `setHostname` `setPort` `setPathname` `setSearch` `setHash` `setQuery` `setFragment`, `join(ref)`, `equals`; query pairs `params` `param` `paramAll` `hasParam` `setParam` `appendParam` `deleteParam` `setParams` `sortParams`; `percentEncode(s, EncodeSet)`, `percentDecode`, `percentDecodeBytes`, `formEncode`, `formDecode`, `parseHost`, `domainToAscii`, `domainToUnicode`, `punycodeEncode`, `punycodeDecode`; see [below](#url) |
| `std.db.sqlite` | SQLite databases through the system's library, loaded at run time, or built into the program with `useBundled()`: `open(path, Options { readOnly, create, uri, busyTimeout, cacheSize, noMutex }) : Db ! SqlError`; `Db.exec(sql, ...)`, `query<T>(sql, ...) : List<T>` (rows into a struct by column name), `rows(sql, ...) : List<Row>`, `prepare(sql) : Stmt`, `begin() : Tx`, `lastInsertId` `changes` `inTransaction` `close`; `Row` (`int` `f64` `str` `blob` `bool` and their `...OrNull` forms by index or name, `value`, `columns`), `Value`, `SqlError { code, message }`; `available()`, `version()`; see [below](#sqlite) |
| `std.media.h264` | H.264 encoding with OpenH264, built into the program (Linux and macOS): `Encoder.new(Options { width, height, fps, bitrate, screen, keyInterval }) : Encoder ! H264Error`, `encode(i420, ms) : Frame ! H264Error` (`data` as Annex B NAL units, `key`, `skipped`), `forceKey`, `width`, `height`; `available()`; see [below](#h264) |
| `std.crypto` | hashes `sha1` `sha224` `sha256` `sha384` `sha512` `sha512t256`, `hash(Hash, bytes)`, `Hasher` (`new(Hash)` `update` `updateStr` `finish` `reset`), BLAKE2b (`blake2b`, `Blake2b`); `hmac`, `Hmac`; `hkdf` `hkdfExtract` `hkdfExpand`; `pbkdf2`; `constantTimeEqual`; AEADs `ChaCha20Poly1305`, `XChaCha20Poly1305`, `AesGcm` (`seal` `open`), `ChaCha20`, `Poly1305`, `Aes`; X25519 (`X25519Key`, `x25519`), Ed25519 (`Ed25519Key`, `ed25519Verify`, ctx and ph), P-256 ECDSA and ECDH (`P256Key`, `P256PublicKey`), RSA PKCS#1 v1.5 and PSS signatures (`RsaPublicKey`, `RsaPrivateKey`); `PrivateKey` (PKCS#8, SEC1, PEM) and `PublicKey` (SubjectPublicKeyInfo, PKCS#1, PEM); `randomBytes(n)` `randomFill` `randomU64` `randomRange` from the OS; see [below](#crypto) |
| `std.crypto.x509` | X.509 certificates: `parse(der)`, `parsePem`, `parsePemAll` → `Certificate` (`subject` `issuer` `notBefore` `notAfter` `dnsNames` `ipAddresses` `isCa` `pathLen` `keyUsage` `extKeyUsage` ..., `publicKey()` `spkiSha256()` `fingerprint()` `encodePem()`); `verify(leaf, intermediates, Roots, VerifyOptions { name, time, purpose })` (chain building to trust anchors, signatures, validity, path length, key usage, name constraints), `verifyName(cert, hostOrIp)` (RFC 6125); `Roots` (`new` `fromPem` `add` `addPem`), `systemRoots()` (SSL_CERT_FILE/SSL_CERT_DIR, the Linux CA bundle, macOS Security.framework, the Windows ROOT store); `Template` with `selfSigned(t, key)` and `issue(t, pub, issuer, issuerKey)`, `csr(subject, dnsNames, key)` and `parseCsr` (PKCS#10); see [below](#x509-certificates) |
| `std.crypto.minisign` | minisign signatures (minisign, rsign, Tauri's updater): `PublicKey.parse(text)` (the file, its base64, or the bare key line), `fromBase64`, `keyIdHex` `toBase64` `encode` `verify(message, sig, allowLegacy = true)` `verifyPrehashed(digest, sig)`; `Signature.parse(text)` (the file or its base64) with `untrustedComment` `prehashed` `keyId` `signature` `trustedComment` `globalSignature`, `signedBy(key)` `encode`; `SecretKey.generate()`, `fromEd25519(key, keyId)`, `parse(text)` (unencrypted, `minisign -W`), `encode` `publicKey` `sign(message, trustedComment, prehash = true)` `signDigest`; `verify(keyText, message, sigText) : Signature ! MinisignError`; see [below](#minisign) |
| `std.encoding.der` | ASN.1 DER: `Reader` (`readSequence` `readInt` `readUnsigned` `readOid` `readBitString` `readOctetString` `readString` `readExplicit` ... `finish`), `Element`, `Writer` (`begin` `end` `writeInt` `writeOid` ... `take`), tag constants, `DerError`; see [below](#der-and-pem) |
| `std.encoding.pem` | RFC 7468 armor: `encode(label, bytes)`, `decode(text) : Block ! ParseError`, `decodeAll(text)`; `Block { label, bytes }` |
| `std.encoding.hex` | `encode(bytes) : str` (lowercase), `encodeUpper`, `decode(text) : List<u8> ! ParseError` |
| `std.encoding.base64` | RFC 4648: `encode` (standard, padded), `encodeRaw` (no padding), `encodeUrl`, `encodeUrlRaw`, `encodedLen(n, pad)`, `decode(text) : List<u8> ! ParseError` (any of the four); constant-time in the values of the bytes, for PEM private keys |
| `std.uuid` | `v4()`, `v7()`, `parse(text) : Uuid ! ParseError`, `nil()`, `max()`; `Uuid` (`str` `bytes` `fromBytes` `version` `unixMillis` `isNil`, `==`, `<`, hashable, `{id}` and `{id:X}`) |
| `std.io.compression.lz4` | the LZ4 block format: `compress(bytes)`, `decompress(block, size) : List<u8> ! DecodeError`, `maxCompressedSize(n)` |
| `std.io.compression.deflate` | deflate (RFC 1951) with zlib and gzip framing: `compress(bytes, level = 6, format = Format.Raw)`, `decompress(bytes, format) : List<u8> ! DecodeError`, `decompressMax(slice, limit, format, sizeHint) : List<u8>? ! DecodeError` (null past limit bytes); streams with `Inflater` (`new(format, keepRest = false)`, `write(chunk) : List<u8> ! DecodeError`, `finish`, `done`, `rest`) and `Deflater` (`write(chunk) : List<u8>`, `finish`); `crc32` `crc32Update` `crc32Raw` `adler32` `adler32Update` `useCrcFold`; see [below](#deflate-zlib-and-gzip) |
| `std.io.compression.zlib` | `compress(bytes, level = 6)`, `decompress(bytes) : List<u8> ! DecodeError` (RFC 1950) |
| `std.io.compression.gzip` | `compress(bytes, level = 6)`, `decompress(bytes) : List<u8> ! DecodeError` (every member), `decompressMax(bytes, limit) : List<u8>? ! DecodeError`, `isGzip(bytes)` (RFC 1952) |
| `std.io.archive.zip` | zip archives: `Reader.open(bytes) : Reader ! ZipError` (`entries`, `len`, `find(name)`, `read(i, maxBytes)`, `readName`), `Entry` (`name` `kind` `method` `size` `compressedSize` `crc` `mode` `mtime`), `Writer` (`add(name, bytes, method, mode)`, `addDir`, `addSymlink`, `finish`), `isZip`; stored and deflate, CRC-32 checked, zip64; see [below](#zip-and-tar-archives) |
| `std.io.archive.tar` | tar archives: `Reader.open(bytes)`, `Reader.openGz(bytes, maxBytes)` (`entries`, `len`, `find`, `read(i)`, `readName`), `Entry` (`name` `kind` `size` `mode` `mtime` `linkName` ids), `Writer` (`add`, `addDir`, `addSymlink`, `addHardlink`, `finish`, `finishGz`); ustar, pax and GNU long names |
| `std.io.archive` | `cleanPath(name) : str ! PathError`: an entry name as a relative path, absolute, drive and `..` paths refused; `decodeName(bytes, cp437)` |
| `std.gfx.image` | `Image.new(w, h)`, `img[x, y]`, `normalAt`, `savePpm`, `Color.gray/rgbf/new`; PNG: `Image.decodePng(bytes)`, `Image.loadPng(path) : Image ! PngError`, `encodePng(color = PngColor.Auto, level = 6)`, `savePng(path)`; `rgba()`, `Image.fromRgba(w, h, bytes)`; see [below](#png) |
| `std.gfx.model` | 3D models, as Assimp reads them: `load(path, Options { format, steps, obj, fbx, stl }) : Scene ! ModelError`, `loadBytes`, `process(ref scene, Steps)`; FBX 7.1-7.7 (binary and ASCII), OBJ with MTL, STL (ASCII and binary); `Scene` (`nodes` `meshes` `materials` `textures` `animations` `cameras` `lights` `meta`, `findNode` `globalTransform` `pose` `skin` `bounds`), `Mesh`, `Material`, `Bone`, `MorphTarget`, `Animation`; the 32 post-processing steps (`Steps { triangulate, genSmoothNormals, calcTangentSpace, joinIdenticalVertices, ... }`, `Steps.realtimeQuality()`); `save`, `encodeObj`, `encodeStl`, `encodeStlAscii`; see [below](#model) |
| `std.encoding.json` | JSON (RFC 8259): `parse(text) : Value ! JsonError`, `parseBytes`, `stringify(v)`, `pretty(v, indent)`, `object()`, `array()`; `Value` (`Null` `Bool` `Int` `Float` `Str` `Array` `Object`) with `v["key"]`, `v[i]`, `for x in v`, `bool` `int` `float` `str` (each `T?`), `isNull` `isInt` `isNumber` ... `kind` `len` `has` `keys` `items` `members` `set` `push` `remove` `toJson`, and to move a subtree out instead of copying it `take` (leaving Null) `takeKey(key) : Value?` `takeAt(i) : Value?` (leaving Null) `takeItems` `takeMembers`; `equals` `get` `pointer` (RFC 6901); `encode(value)`, `encodePretty`, `decode<T>(text) : T ! JsonError`, `decodeBytes<T>`, `toValue`, `fromValue<T>`, `@serde(...)` attributes (rename, renameAll, alias, skip, skipIfNull, skipIfEmpty, default, flatten, denyUnknownFields; enums with data externally, internally (`tag`), adjacently (`tag`, `content`) tagged or `untagged`), `RawValue`; `Pull` (a streaming pull parser fed in pieces), `Writer` (with `drain` for streamed output) and `Reader`; see [below](#json) |
| `std.window` | `open(Options { title, width, height, resizable, x, y, center, fullscreen, maximized, clientTitleBar }) : Window ! WindowError`; `for e in w` gives `Event`s (`Close`, `Resize`, `KeyDown(Key, repeat, scancode)`, `KeyUp(Key, scancode)` (scancode: the evdev code, the same on every OS: PC set 1 for the main block), `MouseMove`, `MouseDelta(dx, dy)` (while the pointer is locked), `MouseDown`/`MouseUp(MouseButton)`, `Scroll`, `Focus`, `Scale(f32)`, `State(WindowState)`, `Text(str)` and an input method's `Preedit(str, caret)` after `textInput(true)`); `width` `height` `scale()` (the content scale, DPI / 96: Xft.dpi, Wayland's buffer scale, GetDpiForWindow, backingScaleFactor) `isFocused()` `state()` (`Normal` `Minimized` `Maximized` `Fullscreen`) `minimize` `maximize` `restore` `setFullscreen(Fullscreen) : bool` (`Off`, `Borderless`, `Exclusive(VideoMode)`) `fullscreen()` `monitors() : List<Monitor>` `monitor() : Monitor?` (`name`, `x` `y` `width` `height`, `workX` `workY` `workWidth` `workHeight`, `scale`, `primary`, `mode`, `modes`: `VideoMode { width, height, refresh }`) `position() : (int, int)?` `setPosition(x, y) : bool` `center()` `setSize(w, h)` (Wayland can't place windows or switch modes) `setPointerLock(bool) : bool` `isPointerLocked()` `warpPointer(x, y) : bool` (Wayland only while locked) `setTitle` `textInput` `setTextInputRect(x, y, w, h)` `setCursor(Cursor)` (`Arrow` `IBeam` `Hand` `Crosshair` `ResizeEW` `ResizeNS` `ResizeNWSE` `ResizeNESW` `ResizeAll` `NotAllowed` `Wait` `Hidden`) `cursor()` `setClipboard(text)` `clipboard() : str?` `setAccessTree(AccessTree)` (accessibility: `AccessNode { id, role, name, value, x, y, width, height, children, actions, focusable, checked, expanded, selected, range, caret }`, requests as `Event.Access(node, AccessAction, value)`; AT-SPI on Linux, UI Automation on Windows, NSAccessibility on macOS) `native` `close`; a client-drawn title bar (`clientTitleBar`: the program draws the bar and its window buttons and gives the window their places each frame with `setTitleBar(TitleBar { caption, minimize, maximize, close, exclude })` (`HitRect { x, y, width, height }` in pixels), answered to the OS's hit test (`hitTitleBar`, `titleBarHit(x, y) : TitleBarHit`): on Windows `WM_NCCALCSIZE`/`WM_NCHITTEST` with the system frame, Aero Snap and Snap Layouts on the maximize button, on X11 `_MOTIF_WM_HINTS` and `_NET_WM_MOVERESIZE`, on Wayland `xdg_toplevel.move`/`resize`; `hasClientTitleBar()` is false on macOS, which keeps its own; the edge bands show the edges' resize cursors on X11 and Wayland; `requestClose()` reports `Close`; `setBorderColor(BorderColor) : bool` (`Default`, `None`, `Rgb(r, g, b)`: the border Windows 11 draws, `DWMWA_BORDER_COLOR`; false and nothing elsewhere); see [graphics.md](graphics.md#client-drawn-title-bar)). Wayland (xdg-shell, with libdecor or server-side decorations) or X11 (xcb) on Linux, picked at run time (`BISMUTH_WINDOW=x11\|wayland` overrides), Win32, Cocoa; input methods on Wayland, Windows, macOS and X11 (IBus over D-Bus), dead keys and compose sequences on X11 and Wayland; see [graphics.md](graphics.md) |
| `std.window.x11` | the X server's screen and input, for screen sharing and remote control: `open(Options { display, shm }) : Display ! X11Error`; `Display.capture(x, y, w, h, ref frame, PixelFormat)` (MIT-SHM, else GetImage; RGBA or BGRA) `captureFrame`, `moveTo` `moveBy` `button` `click` `scroll` `key` `keysym` `typeKeysym` (XTEST), `keycodeOf` `keysymOf`, `pointer` `isKeyDown`, `cursor() : Cursor` (XFIXES), `width` `height` `refreshSize` `sync` `close`; libxcb loaded at run time; see [below](#x11-capture-and-input) |
| `std.gamepad` | `open(Options { deadzones, devices, mappings }) : Gamepads ! GamepadError`; `for e in pads` gives `Event`s (`Connected(pad)`, `Disconnected`, `ButtonDown(pad, Button)`, `ButtonUp`, `AxisMove(pad, Axis, value)`); `down` `axis` `leftStick` `rightStick` `state` `rawState` `count` `list` `name` `guid` `rumble` `canRumble` `setDeadzones` `addMappings` `addVirtual` `setVirtual` `removeVirtual` `update` `close`; `radial`, `linear`, `parseMapping`. evdev on Linux, XInput on Windows, not yet on macOS; see [below](#gamepad) |
| `std.audio` | `open(Options { backend, rate, channels, sample, periodFrames, bufferFrames, name, device }) : Device ! AudioError`, `offline(rate, channels, sample)`, `openWith(backends, opts)`; `Device.play(sound, PlayOptions { volume, pan, pitch, looping, interp, paused }) : Voice`, `queue(channels, rate) : Stream`, `stream(channels, rate, feed) : Stream`, `setVolume` `stopAll` `playing` `format` `backend` `frames` `error` `render(frames)` `output()` `close`; `Voice`: `stop` `pause` `resume` `setVolume` `setPan` `setPitch` `setLoop` `seek` `position` `isPlaying`; `Stream`: `push(samples)` `queued` `finish` and the voice methods; `playStream(decoder, opts) : Stream`; `Sound.fromSamples`, `decodeWav`, `decodeFlac`, `decodeMp3`, `decodeVorbis`, `decode`, `load`; `decoder(bytes)`, `openDecoder(path) : Shared<Decoder> ! AudioError`, `Decoder`: `read(ref out, frames)` `rewind` `readAll` `channels` `rate` `frames` `kind`; `Backend` (extend it for a new output), `Offline`. PulseAudio then ALSA on Linux, WASAPI on Windows; see [below](#audio) |
| `std.gfx` | `open(window, Options { backend, vsync, depth, debug, samples, parallel, timing }) : Device ! GfxError`, `headless(Options)`; objects from descriptors: `buffer` (`.Vertex`, `.Index`, `.Storage`; `indirect`) `image` (`mipLevels`, `samples`, `kind: ImageKind` `.D2`/`.Cube`/`.D2Array` with `layers`, `storage`: written by compute stages as a `StorageImage2D<T>`, `storageTexel(format)`) `sampler` (`mipFilter`, `compare`, `minFilter` `magFilter` `wrapU` `wrapV` `wrapW` `border` `maxAnisotropy` `lodBias` `minLod` `maxLod`), `supports(format, renderTarget)`, formats from `Rgba8` to `Bc7` (`rowBytes` `imageBytes`) `shader` `pipeline` `pipelineWith` `computePipeline` `bindings` (`storage`), with `update` `updateImage` `updateImageRegion(img, ImageRegion { x, y, width, height, level, layer }, data, stride)` `copyImage` `generateMipmaps` `readImage` `readImageRegion` `readBuffer`, asynchronous `readImageAsync` `readBufferAsync` (`Readback`: `readbackReady` `takeReadback` `destroyReadback`) and `destroyBuffer` … `destroyBindings`; `beginPass(PassDesc { clear, action, depthAction, color, colors, clears, depth, resolve, sort, layer })` (depth alone: a depth-only pass; `colors`: several targets; `layer`: a cube face or array layer) `beginCompute` `encoder` `endPass` `commit`; `Encoder`: `setPipeline` `setBindings` `setUniforms` `setUniformBytes` `setVertexBuffer` `setIndexBuffer` `setScissor` `setViewport` `setBlendConstant` `draw` `drawIndexed` (both with `firstInstance`) `dispatch` `drawIndirect` `drawIndexedIndirect` `dispatchIndirect` (`DrawArgs` `DrawIndexedArgs` `DispatchArgs`) `beginQuery` `endQuery` `setSortKey`; occlusion queries (`occlusionQuery() : Query`, `queryResult(q) : i64?`, `destroyQuery`); pipelines' `blend` presets or any `BlendState` (`BlendFactor`, `BlendOp`), `writeMask` (`WRITE_RGB`, ...), `colorTargets` (`ColorTarget`, several targets), `alphaToCoverage`, `depthBias` `depthBiasSlope` `depthBiasClamp`, `vertexLayout` (`VertexLayout { buffers: VertexBufferLayout { stride, instance }, attrs: VertexAttr { location, buffer, offset, format } }`, `VertexFormat` from `Float` to `Half4` and `Unorm1010102`); `bytes(list)`, `rgb` `rgba`, `stats` (`gpuMs` with `Options.timing`); built-in shaders `colorVs`/`colorFs`, `spriteVs`/`spriteFs`, `uiVs`/`uiFs`, `mipVs`/`mipFs`/`mipArrayFs`. Vulkan (Linux on Wayland or X11, Windows), Direct3D 12 and 11 (Windows), Metal (macOS), loaded at run time; see [docs/graphics.md](graphics.md#the-api-as-built-16) |
| `std.ui` | an immediate-mode GUI: `new()`, `Ui` with `beginFrame` `endFrame : DrawData` `feed(window.Event)`, windows, layout, `label` `button` `checkbox` `sliderInt` `sliderFloat` `textField` `progressBar` `collapsingHeader` `treeNode` tables, `gpu(dev)` (draw through `gfx`), `render` (CPU); TrueType text; SVG icons (`loadIcon`, `iconButton`, `drawIcon`); see [below](#ui) |
| `std.webview` | the system's web view in a native window, for apps whose interface is a web page: `app() : App ! WebviewError`, `App.open(Options { title, width, height, minWidth, minHeight, resizable, decorations, visible, center, background, dataDir, url, html, initScript, devtools, allowNavigation }) : Window`, `run(fn(Event))`, `quit`, `proxy() : Proxy` (`post(data)`, `quit`, from any thread), `windows`; `Event`s `Message(w, text)` (the page's `window.ipc.postMessage`), `CloseRequested`, `Moved`, `Resized`, `Focused`, `Loaded`, `Destroyed`, `User(data)`; `Window`: `navigate` `loadHtml` `eval` `url` `setTitle` `show` `hide` `focus` `minimize` `maximize` `unmaximize` `toggleMaximize` `isMaximized` `isMinimized` `isVisible` `startDrag` `position` `size` `setPosition` `setSize` `setMinSize` `geometry` `setGeometry` `destroy`, `nativeHandle() : NativeHandle { kind, window, webview, xid, surface }` (the HWND and ICoreWebView2Controller, NSWindow and WKWebView, or GtkWindow and WebKitWebView with the X11 window id or Wayland surface; `NATIVE_WIN32` `NATIVE_COCOA` `NATIVE_GTK`). WebKitGTK 4.1 on Linux, WebView2 on Windows (no WebView2Loader.dll), WKWebView on macOS, all loaded at run time; see [below](#webview) |
| `std.desktop` | what desktop apps share: `configDir(appId)` (`$XDG_CONFIG_HOME/<appId>`, `%APPDATA%\<appId>`, `~/Library/Application Support/<appId>`), `configBase()`, `validAppId(id)`, `tauriOs()` `tauriArch()` (Tauri's platform names), `findProgram(name)`, `parentDir` `baseName`, `SEP`; the services are its submodules below; see [below](#desktop) |
| `std.desktop.dialog` | native file and folder pickers and message boxes, blocking, for the UI thread: `openFile(Options { title, directory, fileName, filters, multiple, parent }) : List<str> ! DialogError` (empty: cancelled; `parent`, a `webview.Window?`, owns the dialog, else the thread's active window), `openFolder`, `saveFile : str?`, `message(title, text, Level, parent)`, `confirm : bool` (OK/Cancel), `ask : bool` (Yes/No); `Filter { name, extensions }`, `Level.Info` `Warning` `Error`; IFileOpenDialog and MessageBoxW on Windows, NSOpenPanel and NSAlert on macOS, the xdg-desktop-portal FileChooser over D-Bus on Linux, else zenity, kdialog or GTK 3 |
| `std.desktop.single` | one instance per user: `acquire(appId, Options { args, timeout }) : Instance ! SingleError`, `Instance.Primary(p)` (`p.messages() : Chan<Message>`, `p.forward(proxy)` to a web view as `Event.User`, `p.release()`) or `Instance.Secondary` (the primary has this launch's `Message { args, cwd }`: exit); `decode(userData) : Message?`; an flock'd lock file and a Unix socket, or a named mutex and pipe on Windows |
| `std.desktop.windowstate` | a window's place kept across runs in Tauri's `.window-state.json`: `load(appId, label) : webview.Geometry?` (fitted onto the screens), `save(appId, label, g)`, `loadFrom(path, label)`, `saveTo`, `fit(g)`, `stateFile(appId)`; `track(window, appId, label) : Tracker` restores it and `Tracker.handle(event)` saves it (on CloseRequested, and while it moves) |
| `std.desktop.opener` | `openUrl(url, schemes = ["http", "https", "mailto"]) : void ! OpenError` (other schemes refused unless named), `openPath(path)`, `reveal(path)` (the folder, with it selected), `urlAllowed(url, schemes)`, `schemeOf(url)`; xdg-open or gio, open(1), ShellExecuteW |
| `std.desktop.updater` | self-update from Tauri's updater manifests (static `latest.json` with `platforms`, or a dynamic server's one build, 204 for none): `check(Config { endpoints, pubkey, currentVersion, target, installer, headers, timeout, maxBytes, installMode, beforeExit }) : Update? ! UpdateError` (async), `Update` (`version` `notes` `date` `target` `url` `signature`; `download(progress) : List<u8>` verified with minisign, `verify`, `install(bytes)`, `downloadAndInstall`), `restart()`; `parseVersion` `compareVersions` (SemVer), `parseRelease`, `pick`, `targets`, `endpointUrl`, `verifyBytes`; an AppImage on Linux, a `.app.tar.gz` on macOS, NSIS or MSI on Windows |
| `std.text.regex` | regular expressions in linear time: `compile(pattern) : Regex ! RegexError`; `isMatch` `find` `findAt` `findAll` `captures` `capturesAt` `capturesAll` `replace` `replaceAll` `replaceN` `replaceAllWith` `split` `splitN` `groupCount` `groupNames` `groupIndex` `fresh`; `Match`, `Captures`, `escape`; see [below](#regex) |
| `std.text.unicode` | Unicode normalization (UAX #15) for `str`: `normalize(s, Form)` (`Form.Nfc` `Nfd` `Nfkc` `Nfkd`), `nfc` `nfd` `nfkc` `nfkd`, `isNormalized(s, Form)`, `quickCheck(s, Form) : QuickCheck` (`Yes` `No` `Maybe`); `combiningClass(char) : int`, `isCombiningMark(char)` (Mn, Mc, Me), `decompose(char, compat)`; `UNICODE_VERSION`; see [below](#unicode-normalization) |
| `std.term.vt` | a headless terminal emulator (what the Rust vt100 crate does): `Parser.new(rows, cols, scrollback)` with `process(bytes)` `processStr` `resize` `takeReplies` and its `screen`; `Screen`: `contents` `contentsWithScrollback` `contentsBetween` `rowText` `cell(row, col) : Cell?` `cursorPosition` modes, title, scrollback rows, `resize`; redraw and diff output `contentsFormatted` `contentsDiff(prev)` `stateFormatted` `stateDiff(prev)`; `Machine` (the escape-sequence state machine), `charWidth`, `strWidth`; see [below](#vt) |
| `std.interop.dl` | `open(name) : Lib?`, `c()` (the C runtime), `Lib.sym(name) : *u8`, `has`, `close`: shared libraries and their symbols at run time, for `extern c fn` pointer types |
| `std.interop.com` | COM objects through their vtables, and COM objects made in Bismuth: `Guid` (`str` `lower` `bytes` `isNull`, `==`, prints as `{6B29FC40-...}`), `guid(...)`, `parseGuid`, `guidFromBytes`, `GUID_NULL`, `IID_IUNKNOWN` `IID_IAGILEOBJECT` `IID_IINSPECTABLE` `IID_IMARSHAL` `IID_ICLASSFACTORY`; `method<T>(obj, slot)`, `addRef`, `release`, `releaseCount`, `query`, `queryHr`, `cast : *u8 ! HrError`, `Ptr` (an owned reference: `attach` `borrow` `raw` `detach` `method` `cast` `query`); HRESULTs `failed` `succeeded` `hex` `hrName` `message` `hrText` `fromWin32` `check(hr, call) : void ! HrError`, `HrError { call, hr }`, `S_OK` ... `RPC_E_CHANGED_MODE`; `implement<T>(iids, methods, state) : *u8`, `state<T>(obj)`, `refs`, `delegate<T>(iid, state, invoke)`; on Windows `init(model)`, `create(clsid, iid, ref hr, context)`, `newObject : Ptr ! HrError`, `newGuid`, `CLSCTX_*`, `COINIT_*`, `taskAlloc<T>`, `takeString`, `taskString`, `CoTaskMemAlloc` `CoTaskMemFree`, `bstr` `fromBstr` `freeBstr`, `Variant` (`ofBool` `ofI4` `ofU4` `ofI8` `ofU8` `ofF64` `ofStr` `ofUnknown`, `kind` `asBool` `asInt` `asF64` `asStr` `asUnknown` `clear` `copy` `convert` `text`), `VT_*`, `lastErrorHr`, `CoInitializeEx`, `CoUninitialize`. See [below](#com-and-winrt) |
| `std.interop.winrt` | the Windows Runtime as COM: `HString` (`new` `attach` `raw` `detach` `str` `len` `compare`), `hstringText`, `takeHString`, `init`, `uninit`, `factory(className, iid)`, `activate(className)`, `className(obj)`, `iids(obj)`, `close` (IClosable), `addHandler` `removeHandler` (`EventToken`), `vectorSize` `vectorItems<T>` `vectorStrings` (IVectorView); on every OS `pinterfaceIid(generic, args)`, `pinterfaceSignature`, `iidFromSignature`, `classSignature` `structSignature` `enumSignature` `delegateSignature` `interfaceSignature`, `SIG_*`, `IID_IVECTORVIEW` `IID_IMAP` `IID_TYPEDEVENTHANDLER` ... `IID_ICLOSABLE`. See [below](#com-and-winrt) |
| `std.interop.objc` | the Objective-C runtime (macOS): `load` `cls` `sel` `send<T>` `sendStret<T>` `msg0` `msg1` `msgU` `msgB` `msgRetU` `msgRetF` `call` `nsString` `toStr` `subclass` `setTag` `tag` `poolPush` `poolPop` |
| `std.collections` | `Set<T>`, `Deque<T>`, `Queue<T>`, `Stack<T>`, `PriorityQueue<T>`, `LinkedList<T>`, `SortedMap<K, V>`, `SortedSet<T>`, `BitSet`, `Counter<T>`, `MultiMap<K, V>`; see [below](#collections) |
| `std.encoding.csv` | CSV (RFC 4180): `parse(text, Options { delimiter, quote, header, mode, crlf }) : Reader`, `fromFile(file, opts)`, `readAll(text, opts) : List<List<str>> ! CsvError`; `Reader` (`read` `next` `err` `header` `column` `line`), `Row` (`row[i]` `get` `field(name)` `len` `line` `toList`), `Field` views (`str` `eq` `parseInt` `parseFloat` `len`); `writer(opts) : Writer` (`field` `value` `endRow` `writeRow` `str` `take` `flush`); see [below](#csv) |
| `std.encoding.toml` | TOML 1.0: `parse(text) : Table ! TomlError`, `encode(table) : str`; `Table` (`get` `at(path)` `set` `has` `getStr` `getInt` `getFloat` `getBool` `getDateTime` `getArray` `getTable` `keys`, `for k, v in t`, `equals` `deepCopy`), `Array`, `Value` (`String` `Integer` `Float` `Boolean` `DateTime` `Array` `Table`; `asStr` ... `typeName` `str`), `DateTime` on `std.time`'s `Date` and `Time`; see [below](#toml) |
| `std.encoding.yaml` | YAML 1.2: `parse(text, Options) : Value ! YamlError` (the first document), `parseAll`, `stringify(v)`, `stringifyAll`, `stringifyFlow`; `encode(value)`, `decode<T>(text)`, `decodeAll<T>`, `decodeValue<T>` (typed, with `@serde` attributes, as json's); `Value` (`Null` `Bool` `Int` `Float` `Str` `Seq` `Map` `Tagged`) with `v["key"]`, `v[i]`, `bool` `int` `float` `str` `tag` `untagged` `kind` `keys` `items` `entries` `set` `push` `equals`; events: `Parser` (`next() : Event`), `parseEvents`, `Event` (`kind` `anchor` `tag` `value` `style` `flow` `implicit` `line` `col`), `eventText`; `Emitter` (`emit(event)`, `value(v)`, `doc(d)`, `take`); `load(text) : List<Doc>` (documents as node graphs); `Options { maxAliasNodes, maxDepth }`; see [below](#yaml) |
| `std.linq` | lazy queries: `xs.query().where(...).orderBy(...).select(...).toList()` over List, Slice, Map and every collection; see [below](#linq) |

## collections

`import std.collections;` then `collections.Set<int>()` (or
`import std.collections as c;`). The types are written in Bismuth on `mem.*`, as List
and Map are, and keep their discipline: mutating calls move elements in,
every element is dropped exactly once (when it is removed, replaced, or
the collection is dropped or cleared), copying a collection copies its
elements with their own copy glue, and reads (`get`, `front`, `first`,
iteration) return copies. Changing a collection while iterating it is not
allowed. `==` on two of these collections compares their storage; use
`equals` to compare elements. (`==` on List and Map compares elements:
a List's in order, a Map's entries in any order.) They live outside the
prelude to keep programs that don't use them compiling fast. Names under
`std` are reserved, so a `collections.bi` beside your program is a module
of its own (`import collections;`) and never shadows this one.

| Type | What | Methods |
|---|---|---|
| `Set<T>` | hash set (open addressing, `core.hash` and `==`, like Map's keys); `for x in s` in table order | `new`/`Set<T>()` `fromList` `len` `isEmpty` `add` (reports if new) `remove` (reports if present) `has` `contains` `addAll` `union` `intersect` `difference` `symmetricDifference` `isSubset` `isSuperset` `isDisjoint` `equals` `toList` `iter` `clear` `clone` |
| `Deque<T>` | ring buffer, O(1) at both ends and by index; `dq[i]`, `for i, x in dq` | `new` `withCapacity` `fromList` `len` `isEmpty` `capacity` `pushBack` `pushFront` `popFront` `popBack` `front` `back` `get` `set` `contains` `toList` `iter` `clear` `clone` |
| `Queue<T>` | FIFO on a Deque; iterates oldest first | `new` `len` `isEmpty` `push` `pop` `peek` `contains` `toList` `iter` `clear` `clone` |
| `Stack<T>` | LIFO on a List; iterates bottom first; `asSlice` gives every Slice method | `new` `len` `isEmpty` `push` `pop` `peek` `contains` `asSlice` `toList` `iter` `clear` `clone` |
| `PriorityQueue<T>` | binary heap: `new()` pops the smallest (by `<`), `maxFirst()` the largest, `withComparator((a, b) => ...)` by any "a pops first" test; iteration is heap order | `new` `maxFirst` `withComparator` `fromList` (O(n) heapify) `len` `isEmpty` `push` `pop` `peek` `contains` `toList` `toSortedList` `iter` `clear` `clone` |
| `LinkedList<T>` | doubly linked; `for x in ll`, `for x in ll.iterBack()` | `new` `fromList` `len` `isEmpty` `pushFront` `pushBack` `popFront` `popBack` `front` `back` `get` `set` `insert` `remove` `removeWhere` `contains` `indexOf` `reverse` `splice` (O(1) move of another list) `toList` `iter` `iterBack` `clear` `clone` |
| `SortedMap<K, V>` | ordered map (AVL tree; keys compare with `<` and `==`), O(log n); `for k, v in m` ascending | `new` `len` `isEmpty` `set` `get` `has` `slotOf` `remove` `first` `last` `firstKey` `lastKey` `floor` (≤) `ceiling` (≥) `lower` (<) `higher` (>) `popFirst` `popLast` `range(lo, hi)` ([lo, hi)) `rangeFrom` `rangeTo` `countRange` `iter` `iterDesc` `keys` `values` `height` `clear` `clone`; entries are `(K, V)?`, so `m.floor(k)?.0` is a key |
| `SortedSet<T>` | ordered set on a SortedMap | `new` `fromList` `len` `isEmpty` `add` `remove` `has` `contains` `first` `last` `floor` `ceiling` `lower` `higher` `popFirst` `popLast` `range` `rangeFrom` `rangeTo` `countRange` `iter` `iterDesc` `union` `intersect` `difference` `isSubset` `equals` `toList` `clear` `clone` |
| `BitSet` | set of small non-negative ints, one bit each, growing as needed; `for i in bits` ascending | `new` `withCapacity` `fromList` `capacity` `add` `remove` `toggle` `has` `contains` `count` (members) `len` `isEmpty` `first` `last` `union` `intersect` `difference` `symmetricDifference` (copies) `unionWith` `intersectWith` `differenceWith` `symmetricDifferenceWith` (in place) `isSubset` `equals` `toList` `iter` `clear` `clone`; also `popCount(u64)` `trailingZeros(u64)` |
| `Counter<T>` | counts occurrences; `for x, n in c` | `new` `fromList` `add` `addCount` (n may be negative) `get` (0 if absent) `has` `remove` `len` (distinct) `total` `isEmpty` `keys` `mostCommon(n)` (count descending, then element) `iter` `clear` `clone` |
| `MultiMap<K, V>` | key to a List of values in insertion order; `for k, vs in mm` | `new` `add` `get` (copy, empty if absent) `countOf` `has` `remove` (returns the values) `removeValue` `len` (values) `keyCount` `isEmpty` `keys` `iter` `clear` `clone` |

## linq

`import std.linq;` adds `query()` to `Slice`, `List`, `Map` (pairs `(K, V)`)
and every collection (`SortedMap`, `Counter` and `MultiMap` give pairs;
the iterators `range`, `iterDesc` and `iterBack` have `query()` too), and
starts queries from `linq.range(start, count)`, `linq.rangeStep(start,
end, step)`, `linq.repeat(x, n)`, `linq.empty<T>()`, `linq.from(slice)`
and `linq.fromIter<T, I>(it)` (any struct with `next(ref self) : T?`).

```bismuth
import std.linq;

let names = people.query()
    .where(p => p.age >= 18)
    .orderBy(p => p.city)
    .thenByDescending(p => p.age)
    .select(p => p.name)
    .toList();
for g in people.query().groupBy(p => p.city) { print("{g.key}: {g.len()}"); }
```

A `Query<I, T>` is a struct around an iterator. The lazy operators wrap it
in another struct and return at once; nothing runs until a terminal
operator or a `for x in query` loop pulls elements through, and only as
many as it needs (`first`, `take`, `any` stop early). Since every adapter
is a plain struct and std is compiled with the program, a pipeline such as
`where`/`select`/`take`/`sum` inlines into one loop, with the lambdas
inlined too and no heap allocation per element. Terminal operators push
elements through the adapters (each has a `walk(f)` beside `next()`), which
compiles to the loop one would write by hand: summing a filtered,
mapped million-element List takes the same 1.8 ms as the hand-written
`for`. A `for x in query` loop pulls with `next()` instead, which is
correct but about twice as slow on such tiny bodies; `query.each(x => ...)`
is the fast form.

Elements reach lambdas as views and are copied only where something keeps
them (`toList`, a `for` loop's variable, `distinct`'s Set); the source is
not changed, and what a query holds is dropped with it.

| Kind | Operators |
|---|---|
| Lazy, streaming | `where` `select` `indexed` (`(i, x)` pairs) `selectMany` (lambda returns a List) `flatten` (a query of Lists) `take` `skip` `takeWhile` `skipWhile` `distinct` `distinctBy` (keep a Set of what was seen) `concat` `zip` (pairs, or `zip(other, (a, b) => ...)`) `scan` `chunk(n)` |
| Lazy, buffered on the first pull | `orderBy` `orderByDescending` `thenBy` `thenByDescending` (stable merge sort; `thenBy` exists only on an ordered query) `order` `orderWith(less)` `reverse` `groupBy` (yields `Grouping { key, items }` in order of first appearance) `join(inner, outerKey, innerKey, result)` (inner equi-join; inner read into a Map) |
| Elements | `first` `last` `single` `elementAt` (panic when missing) and `firstOrNull` `lastOrNull` `singleOrNull` `elementAtOrNull`; `first`, `last`, `single` and their `OrNull` forms also take a predicate |
| Aggregates | `count` `count(pred)` `any` `any(pred)` `all` `contains` `sequenceEqual` `sum` `sum(f)` (from `T(0)`: numbers, vectors) `min` `max` `minBy` `maxBy` (`T?`, null when empty) `average` `average(f)` (`f64?`) `aggregate(seed, f)` `aggregate(f)` `joinStr(sep)` `each(f)` |
| Collecting | `toList` `toSet` `toSortedSet` `toMap(key)` `toMap(key, value)` (later keys win) `toLookup(key)` (a `MultiMap`) |

Two lifetimes matter, because neither is checked yet. Lambdas capture
local variables by reference, so a query must run while the variables its
lambdas use are alive: build and consume it in one function, or return
only queries whose lambdas capture nothing. And a query reads its source
in place, so the source must outlive it and stay unchanged while it runs.
A temporary source lives until the end of its statement, or until the
loop ends for a `for` loop's iterable, so `for w in s.split(" ").query()`
and `let n = s.split(" ").query().count();` are fine.
This is also why `selectMany` takes a lambda that returns a List rather
than a query: a query built inside that lambda would hold lambdas
capturing the lambda's own parameter.

## ui

`import std.ui;` is an immediate-mode GUI in the style of Dear ImGui,
written in Bismuth: the interface is code that runs every frame, and a
widget draws itself and says what happened to it in one call. A frame
ends as a `DrawData` (vertices, indices, draw commands): `ui.gpu(dev)`
draws it through `gfx`, and `render` draws it into a `std.gfx.image` on
the CPU. The design is in
[graphics.md §5](graphics.md#std-ui-the-first-slice).

```bismuth
let gui = ui.new();                                   // ui.new(theme: ui.Theme.light(), scale: 2.0)
gui.beginFrame(width, height);
if gui.beginWindow("Settings", ui.Rect.new(20.0, 20.0, 300.0, 200.0)) {
    gui.sliderInt("Volume", ref volume, 0, 100);
    if gui.button("Save") { save(); }
    gui.endWindow();                                  // only when beginWindow returned true
}
let frame = gui.endFrame();
gui.render(frame, ref img);
```

| Area | API |
|---|---|
| Context | `new(theme, scale)`, `withFont(ttf, theme, scale, pixel) : Ui ! FontError`; `Ui` is a handle (copies share the context) |
| Fonts | `addFont(ttf, size, pixel) : int ! FontError`, `addFontFile(path, size, pixel) : int ! FontError`: more faces, each at a size of its own (pixels at scale 1; 0: the theme's `fontSize`), numbered from 1 (`DEFAULT_FONT`, 0, is the context's own), all baking into the one atlas; `pushFont(font)` / `popFont()` (items take the face's line height; pushes are popped by `endFrame`), `font()`, `fontSize()`; `pixel: true` for a pixel font (a bitmap-style face such as ProggyClean at 13): outlines, advances and metrics snapped to whole pixels and the size scaled by whole multiples of the DPI scale, so it stays crisp; `Typeface.new(font, size, pixel)`, `rasterize(font, glyph, scale, snap)` |
| Frames | `beginFrame(width, height, dt)` (dt: seconds since the last frame, 1/60 by default, what hover delays count), `endFrame() : DrawData`, `render(frame, ref image.Image)`, `renderWith(frame, textures: Map<int, image.Image>, ref image.Image)` for frames with images |
| Input | `feed(window.Event)`; or `mouseMove(x, y)`, `mouseDown(b)`, `mouseUp(b)`, `wheel(dy)`, `keyDown(k)`, `keyUp(k)`, `key(k)`, `typeText(s)` |
| The window | `bindWindow(w)`: text fields cut, copy and paste (Ctrl, or Super/Command, + X, C, V) through the window's clipboard, and each `endFrame` turns its text input on while a field is edited, places its input method at the caret (`Preedit` shows underlined in the field) and sets its pointer shape; without one, a clipboard of the context's own, and `wantsTextInput()`, `textInputRect()`, `cursor()` for the program to pass on |
| Accessibility | `accessTree() : window.AccessTree` (a node per widget, given to a bound window each `endFrame`), `setAccessName(name)` for the root, `accessItem(label, role, rect, value, actions) : u64` for items of your own; `Event.Access` through `feed` presses, focuses, sets, steps, expands or collapses in the next frame |
| Pointer shape | `cursor() : window.Cursor`, what the last frame asked for (`IBeam` over text fields, `ResizeNWSE` over a window's grip, else `Arrow`); `setCursor(c)` from an item of your own |
| Clipboard | `setClipboard(text)`, `clipboard() : str` |
| Windows and regions | `beginWindow(title, Rect, WindowOptions) : bool` / `endWindow()` (dragged by the title, collapsed by its arrow, resized by the grip, raised by a click, scrolled by the wheel and scrollbar; `WindowOptions { titleBar, resize, move, collapse, bringToFront }` leave those out); `beginScroll(label, height)` / `endScroll()` |
| Child regions | `beginChild(label, w, h, ChildOptions) : bool` / `endChild()` (always called): a box in the layout with its own scroll (wheel, scrollbar) and clip, `childBg` behind it; `w`, `h` in pixels, 0 for the rest of the line or of the region in view below, negative for that less some; `ChildOptions { border, padding, horizontal, noScrollbar, autoHeight }` (an outline with window padding, padding alone, sideways scrolling with a bottom scrollbar and Shift+wheel or a sideways wheel, wheel only, as tall as its content); `childScroll(label) : vec2`, `setChildScroll(label, vec2)` |
| Placing windows | `setNextWindowPos(x, y, Cond)`, `setNextWindowSize(w, h, Cond)` for the next `beginWindow` (`Cond.Always` every frame, `Once` the first time, `Appearing` when it shows again); `setWindowRect(title, Rect)`; `beginPanel(label, Rect) : bool` / `endWindow()`, a window at a rectangle given every frame with no chrome that a click doesn't raise (a docked panel); `split(label, area, ref at, vertical, least) : (Rect, Rect)`, two sides of a handle the user drags; `beginTabBar(label)`, `tabItem(label) : bool` (the shown tab), `endTabBar()` |
| Popups | `openPopup(id)`, `beginPopup(id) : bool` / `endPopup()` (sized to its items, laid out unseen the first frame, at the mouse and inside the display; a click outside or Escape closes it), `beginPopupModal(name, closable) : bool` (titled, centred, dims and blocks what is under it; open it with `openPopup(name)`), `closeCurrentPopup()` (in a menu, closes the menus it is in), `isPopupOpen(id)`; context menus `beginPopupContextItem(id, button)` (released over the last item; `""` uses its id), `beginPopupContextWindow(id, button, overItems)`, `beginPopupContextVoid(id, button)` |
| Tooltips | `beginTooltip()` / `endTooltip()`, `setTooltip(text)` beside the mouse, taking no input; `setItemTooltip(text)` and `beginItemTooltip() : bool` once the last item has been hovered, the mouse still, for the theme's `hoverDelay`; `isItemHoveredFor(seconds)` |
| Menus | `WindowOptions { menuBar: true }` and `beginMenuBar() : bool` / `endMenuBar()`, `beginMainMenuBar() : bool` / `endMainMenuBar()` (across the display's top, above the windows); `beginMenu(label, enabled) : bool` / `endMenu()` (a bar's menu opens on a click, then by hovering across the bar; a submenu opens on hover, keeping open while the mouse heads for it), `menuItem(label, shortcut, checked, enabled) : bool`, `menuItemToggle(label, shortcut, ref bool checked, enabled) : bool`; in an open menu Up and Down (Home, End) move, Right opens a submenu, Left closes it, Enter or Space picks, Escape closes |
| Combo boxes | `beginCombo(label, preview, heightItems) : bool` / `endCombo()` (a click opens a popup under the field, as wide, `heightItems` rows tall at most and scrolled past that; the arrow keys start from the selected row, Enter picks), `combo(label, ref int current, items, heightItems) : bool` |
| Selectables | `selectable(label, selected, SelectableOptions, w, h) : bool` (`SelectableOptions { spanColumns, allowDoubleClick, dontClosePopups, disabled }`; in a popup, picking one closes it), `selectableMulti(label, index, ref Selection, opts) : bool` (click selects alone, Ctrl+click toggles, Shift+click selects from the anchor), `multiSelectKeys(ref Selection, count) : bool` (Up, Down, Home, End, with Shift a range; Ctrl+A; Escape clears); `Selection { has, set, clear, count, list, selectAll, click(i, ctrl, shift), moveTo }`; `isMouseDoubleClicked()` |
| Number fields | `dragFloat(label, ref f32, speed, lo, hi, format) : bool`, `dragFloat2`/`3`/`4(label, ref vec2/vec3/vec4, speed, lo, hi, format, colors)`, `dragFloatN(label, MutSlice<f32>, ...)`, `dragInt(label, ref int, speed, lo, hi, format)`, `dragInt2`/`3(ref ivec2/ivec3, ...)`, `dragIntN(label, MutSlice<int>, ...)`: dragged across (speed per pixel past a 3-pixel threshold, Shift ×10, Alt ×0.1), kept from `lo` to `hi` when they differ; Ctrl+click or a double click types a number (applied as typed, Enter, Tab or a click elsewhere end it, Escape puts it back); `colors` marks each part with a bar of its colour (axes), `partRects()` is where the parts went. `inputFloat(label, ref f32, step, stepFast, format)`, `inputInt(label, ref int, step, stepFast)`: a text field with - and + buttons (Ctrl: the fast step). Formats are printf's: `"%.3f"`, `"%d"`, `"%.1f deg"`, `"%%"`; `formatNumber(v, format)`, `parseNumber(text, format) : f64?`, `formatDecimals(format)` |
| Colours | `colorEdit3(label, ref vec3, ColorOptions)`, `colorEdit4(label, ref vec4, opts) : bool`: R, G, B (and A) parts dragged from 0 to 255 (`floats`: 0 to 1) and a swatch that opens a picker under it; `colorPicker3`/`4(label, ref vec3/vec4, opts) : bool`: a saturation-value square in the hue, a hue bar, an alpha bar, the swatch and hex code (the hue kept while saturation or value is 0; `nextItemWidth` sizes it); `colorButton(label, vec4, w, h) : bool`; `ColorOptions { noInputs, noPicker, noLabel, floats, noAlphaBar }`; `rgbToHsv`, `hsvToRgb`, `toRgba8`, `colorHex(c, alpha)`, `parseColorHex(text) : vec4?`. A screen reader sees a `ColorWell` valued `#RRGGBB(AA)` and may set it so |
| Tables | `beginTableWith(label, columns, TableOptions) : bool` / `endTable()` (Dear ImGui's tables; false only for a scrolled table out of view): `tableSetupColumn(label, ColumnOptions { width, weight, noResize, noSort, defaultSort, preferDescending })` per column before the first row (a fixed width in pixels, or 0: stretched, sharing what fixed columns leave by weight), `tableHeadersRow()` (or `tableNextRow()` and `tableHeader(label)` per cell), `tableNextRow(minH)`, `tableNextColumn() : bool` and `tableSetColumnIndex(c) : bool` (each cell a region clipped to its column; a row is as tall as its tallest cell), `tableSetBgColor(c)` for the row, `tableSortSpecs() : TableSortSpecs?` (`specs: List<ColumnSort { column, descending }>`, `dirty` since the last call). `TableOptions { resizable, sortable, sortMulti, sortTristate, rowBg, borders, bordersOuter, bordersInnerV, bordersInnerH, scrollY, freezeRows, width, height }`: borders dragged to resize (a double click fits a fixed column to its content), a header click sorts (again: the other way; Shift with `sortMulti`: adds a column; `sortTristate`: a third click unsorts), alternating row colours, rows scrolled within `height` under header rows (and `freezeRows`) that stay put. A selectable with `spanColumns` spans the table's row |
| List clipper | `beginClipper(count, itemH) : (int, int)` / `endClipper()`: in a window, region or table (where each item is a row), the range of the `count` items of `itemH` pixels (0: measured from those shown last frame) that are in view; lay out only those, the rest is stood in for |
| Layout | lines top to bottom; `sameLine()`, `separator()`, `spacing()`, `columns(n)` / `endColumns()`, `beginTable(label, headers)` / `endTable()` (a simple table), `pushId(s)` / `popId()` |
| Widgets | `label(text)`, `labelColored(text, Rgba8)`, `button(label) : bool`, `checkbox(label, ref bool) : bool`, `sliderInt(label, ref int, lo, hi) : bool`, `sliderFloat(label, ref f32, lo, hi, decimals) : bool`, `textField(label, ref str) : bool`, `textArea(label, ref str, height, lineNumbers) : bool` (many lines: Enter and Tab type, Up/Down/PageUp/PageDown by lines, Home/End per line, ctrl+Home/End the whole text, scrolling to the caret, a line-number gutter), `progressBar(fraction, overlay)`, `image(texture, w, h, uv0, uv1, tint, border)`, `imageButton(label, texture, w, h, uv0, uv1, tint) : bool`, `collapsingHeader(label, defaultOpen) : bool`, `treeNode(label, defaultOpen) : bool` / `treePop()` |
| Focus | `setKeyboardFocusHere()`: the next text field or area takes the keyboard with its text selected (this frame or the next); `focusItem(id)` the same for the field with that id |
| State | `id(label)`, `hotId()`, `activeId()`, `focusId()`, `itemRect()`, `itemHovered()`, `windowRect(title) : Rect?`, `windowScroll(title)`, `caret() : (int, int)` |
| Look | `Theme` (colors, padding, spacing, font size; `splitter`, `tab` `tabHover` `tabActive`; `popupBg` `popupBorder` `popupRounding` `menuBarBg` `modalDim` `navHighlight` `hoverDelay`; `tableHeaderBg` `tableHeaderHover` `tableBorder` `tableBorderLight` `tableRowBg` `tableRowAlt` `cellPadX` `cellPadY`; `childBg` `childRounding`; `dragDropTarget` `dragThreshold`; `windowRounding` `frameRounding` `grabRounding` `scrollbarRounding`, `frameBorderSize` `frameBorder`, `windowShadow*` and `buttonShadow*` (size, X, Y, color), `titleText`; `Theme.dark()`, `Theme.light()`), `theme()`, `setTheme(t)`, `pushTheme(t)` / `popTheme()`, `scale()`, `setScale(s)`, `measure(text)`, `lineHeight()`; `hex(0xRRGGBB, alpha)`, `mixColor(a, b, t)`, `withAlpha(c, a)` |
| Drag and drop | after an item, `beginDragDropSource(noPreview) : bool` (held and moved past the theme's `dragThreshold`; a label or image without an id too) then `setDragDropPayload(kind, bytes)`, `setDragDropPayloadText(kind, text)` or `setDragDropPayloadValue<T>(kind, value)` (JSON) and the preview's items (in a tooltip at the mouse), `endDragDropSource()`; after an item, `beginDragDropTarget() : bool` (or `beginDragDropTargetRect(label, rect)`) then `acceptDragDropPayload(kind, beforeDelivery, noHighlight) : Payload?` (in the frame it is dropped, `delivery`; with `beforeDelivery` while it is over the target, `preview`; outlined in `dragDropTarget` meanwhile; nested targets: the smallest takes it), `endDragDropTarget()`; `Payload { kind, data, preview, delivery, text(), value<T>() : T? }`; `dragDropPayload() : Payload?`, `isDragDropActive()` |
| Custom widgets | `reserve(w, h) : Rect`, `interact(label, rect) : Interaction { hovered, pressed, held, clicked }`, `isVisible(r)`, `isHovered(r)`, `contentRect()`, `mousePos()`, `isMouseDown(b)` `isMousePressed(b)` `isMouseReleased(b)`; `beginOverlay(label, rect, input)` / `endOverlay()` above every window (tooltips, dropdowns, popups) |
| Drawing | into the current window, anti-aliased: `fillRect(r, c, rounding, corners)`, `strokeRect(r, c, rounding, thickness, corners)`, `fillRectGradient(r, from, to, horizontal, rounding, corners)`, `fillCircle` `strokeCircle`, `line`, `polyline(points, c, thickness, closed)`, `fillConvex(points, c)`, `fillTriangle`, `shadow(r, c, blur, dx, dy, rounding, spread, hollow)`, `drawText(x, y, text, c)`, `drawImage(texture, r, uv0, uv1, tint)` (texture: an id of yours, 1 or more, that the renderer maps to an image), `pushClip(r)` / `popClip()`; `Corners` (`all` `top` `bottom` `left` `right` `none`, or fields); the same on `DrawList` |
| Output | `DrawData { width, height, vertices: List<gfx.UiVertex>, indices: List<u32>, cmds: List<DrawCmd>, atlasWidth, atlasHeight, atlasVersion }`, `DrawCmd { clip, texture, indexOffset, indexCount }`; `atlas() : Atlas` (`width`, `height`, `pixels` one byte each (the coverage), `colors` three bytes each once a coloured icon is baked (else empty: white), `version`, `rgba()`); `renderFrame(frame, atlas, ref img)`, `imageChecksum(img)` |
| GPU | `gpu(dev, format) : Gpu ! gfx.GfxError` (format: the window's by default, or a render target's); `Gpu.setTexture(id, gfx.Image)` (a texture or render target for `image` and `drawImage`, bilinear), `removeTexture(id)`; `Gpu.upload(gui, frame)` once a frame before the pass, `Gpu.draw(ref enc, frame)` inside it, `Gpu.destroy()`; see `examples/ui` |
| Text | `Font.parse(bytes) : Font ! FontError` (`glyphIndex`, `advance`, `leftBearing`, `box`, `outline`, `scaleForPixelHeight`), `rasterize(font, glyph, scale) : GlyphBitmap`, `Typeface.new(font, size)` (`glyph(cp)`, `measure(s)`, `atlas`), `defaultFontData()`; `utf8Decode`, `utf8Next`, `utf8Prev`, `utf8Write`, `utf8Count` |

A label's text after `##` is not shown, only hashed into the id
(`"Delete##3"`). Ids are unique per window, tree node, table and
`pushId`. Text fields edit a copy while focused and write each change
back: click or Tab to focus, arrows (with ctrl, by word), Home, End,
Backspace, Delete, ctrl+A, shift or a mouse drag to select; Enter or a
click elsewhere ends editing, Escape restores the text from before. Not
yet: kerning, keyboard navigation between widgets other than Tab and
inside popups and menus.

Accessibility nodes: menus are `MenuBar`, `Menu` and `MenuItem` (with
`checked` and `disabled`, the shortcut as the description), popups `Menu`,
a modal popup `Dialog`, a tooltip `Tooltip`, a combo box `ComboBox` (its
preview as the value, expanded while open), a selectable `ListItem`, a
number field (each part of one) `SpinButton` with its range when it has
one, taking `Focus` (to type), `SetValue`, `Increment` and `Decrement`.
A `beginTableWith` table is a `Table` of `Row`s holding their cells'
items; its headers are `ColumnHeader`s valued `ascending` or
`descending` when sorted by, pressed to sort.

The built-in font is a Latin subset of Instrument Sans (SIL Open Font
License, `std/ui/fonts/`). No other face is built in: Dear ImGui's
ProggyClean (MIT licence, Tristan Grimmer) ships with Dear ImGui, and
`addFontFile(path, 13.0, pixel: true)` draws it as Dear ImGui does.

## webview

`import std.webview;` shows a web page in the system's own web view, in a
native window: the shell of a desktop app whose interface is a page (one
its own HTTP server serves on loopback, say), as Tauri and wry make one.
Nothing is linked at build time: WebKitGTK 4.1 and GTK 3 are loaded on
Linux (`libwebkit2gtk-4.1.so.0`; Debian and Ubuntu's
`libwebkit2gtk-4.1-0`), WebView2 on Windows (the installed Evergreen
runtime, found in the registry and started through its
`EmbeddedBrowserWebView.dll` as Microsoft's loader does, so no
`WebView2Loader.dll` ships; `WEBVIEW2_BROWSER_EXECUTABLE_FOLDER` names a
fixed-version runtime; a `WebView2Loader.dll` beside the program is the
last resort), and WKWebView through the Objective-C runtime on macOS.
`app()` reports `NoRuntime` where there is none, or no display.

```bismuth
let app = webview.app()?;
let proxy = app.proxy();
let server = spawn serveThePage(proxy);              // tasks keep running during run()
let w = app.open({
    title: "Sulfur", width: 1320, height: 860, minWidth: 640, minHeight: 420,
    decorations: false, visible: false, background: webview.Color { r: 30, g: 30, b: 30 },
    url: "http://127.0.0.1:47615/", initScript: bridgeJs,
    allowNavigation: url => url.startsWith("http://127.0.0.1:47615/"),
})?;
app.run(e => match e {
    Message(w, text) => match text {
        "ready" => w.show(),
        "drag" => w.startDrag(),
        "minimize" => w.minimize(),
        _ => w.eval("window.reply({json.encode(text)})"),
    },
    CloseRequested(w) => w.eval("window.dispatchEvent(new Event('app-close'))"),   // or w.destroy()
    User(data) => print("from a task: {data}"),
    _ => {},
});
```

| Area | API |
|---|---|
| App | `app() : App ! WebviewError` (once per process; later calls return it), `open(Options) : Window ! WebviewError`, `run(fn(Event) handler)` (on the main thread, until `quit()` or the last window is destroyed), `quit()`, `proxy() : Proxy`, `windows() : List<Window>` |
| Options | `title`, `width` `height` (the content's), `minWidth` `minHeight`, `resizable`, `decorations` (false: no title bar or border), `visible` (false: hidden until `show()`), `center`, `background : Color?` (`Color { r, g, b, a }`), `dataDir` (cookies, storage, caches), `url`, `html`, `initScript` (each new document, top frame, before its own scripts), `devtools`, `allowNavigation : fn(str) : bool` (the navigation filter) |
| Events | `Message(Window, str)`, `CloseRequested(Window)`, `Moved(Window, x, y)`, `Resized(Window, width, height)`, `Focused(Window, bool)`, `Loaded(Window, url)`, `Destroyed(Window)`, `User(str)` |
| Page | `navigate(url)`, `loadHtml(html)`, `eval(script)` (its result is dropped), `url()` |
| Window | `setTitle`, `show`, `hide`, `isVisible`, `focus`, `minimize`, `isMinimized`, `maximize`, `unmaximize`, `toggleMaximize`, `isMaximized`, `startDrag` (a frameless window's own title bar), `destroy`, `isDestroyed` |
| Native | `nativeHandle() : NativeHandle { kind, window, webview, xid, surface }`, the OS objects behind the window for native integration std.webview leaves out (a dialog's parent, taskbar progress, menus): `NATIVE_WIN32` (the HWND; the `ICoreWebView2Controller`), `NATIVE_COCOA` (the `NSWindow`; its `WKWebView`) or `NATIVE_GTK` (the `GtkWindow`; its `WebKitWebView`; `xid`, the X11 window id, under X11; `surface`, the `wl_surface`, under Wayland, where a window not shown yet is realized first). The first two are `std.window`'s numbers for its `Native`. A destroyed window's handle is all null, kind 0; use the objects on the UI thread |
| Geometry | `position() : (int, int)` (the outer frame's top-left), `size() : (int, int)` (the content's), `setPosition`, `setSize`, `setMinSize`, `geometry() : Geometry { x, y, width, height, maximized }` (where the window is when not maximized, to save), `setGeometry(g)` |
| Threads | `Proxy.post(data)` delivers `User(data)` on the UI thread, in order, from any thread or task; `Proxy.quit()` |

The page's side is `window.ipc.postMessage(text)` on every OS, defined by
an initialization script before the page's own (over
`window.webkit.messageHandlers.ipc` in WebKit and
`window.chrome.webview.postMessage` in WebView2); each call is a
`Message` event, and the host answers with `eval`. A window never closes
by itself: the close button (or Alt+F4, or Cmd+Q) is `CloseRequested`,
and the host calls `destroy()` when it agrees. Events come one at a time
on the UI thread; one that a handler causes waits until it returns.
`run()` blocks the main thread in the OS's event loop (gtk_main, a Win32
message loop, `[NSApp run]`) while tasks run on the scheduler's worker
threads; proxies wake it with `g_idle_add`, a posted window message, or
`dispatch_async_f` on the main queue. Positions and sizes are logical
pixels (96 DPI on Windows, points on macOS, GTK's on Linux); Wayland
tells no window its position, so it is 0, 0 there. New windows the page
asks for (`window.open`, `target=_blank`) are refused: the page asks the
host to open the link instead. On Linux the navigation filter also sees
iframes' navigations; elsewhere only the top frame's.

Per OS: on Windows the main thread joins a single-threaded COM apartment,
and a window without decorations keeps invisible resize borders on its
sides and bottom (its top edge does not resize); the data folder defaults
to `%LOCALAPPDATA%\<program>\WebView2`. On macOS a data folder needs
macOS 14 (a `WKWebsiteDataStore` per folder, by an identifier made from
its path; WebKit chooses where it lives), and the app gets a main menu
with Quit and the Edit commands. The macOS backend type-checks but has not
yet run on a Mac. `examples/webview` is a complete shell: a page served by
`std.net.http` from a task, messages both ways, a proxy, the window
commands, and a `--test` mode that exits by itself (under `xvfb-run` on
Linux).

## desktop

`std.desktop` has the shell services a desktop app needs beside its
window, what Tauri's dialog, single-instance, window-state, opener and
updater plugins give a Tauri app. With `std.webview` they make the same
kind of app: [`examples/desktop`](../examples/desktop/main.bi) uses all
of them around one web view.

```bismuth
import std.desktop.dialog;
import std.desktop.opener;
import std.desktop.single;
import std.desktop.updater;
import std.desktop.windowstate;

const APP_ID = "dev.example.demo";

fn main() {
    let p = match single.acquire(APP_ID).expect("single instance") {
        Secondary => os.exit(0),            // the running instance has our arguments
        Primary(p) => p,
    };
    let app = webview.app().expect("a web view");
    p.forward(app.proxy());
    let w = app.open({ title: "demo", url: url, visible: false }).expect("a window");
    var place = windowstate.track(w, APP_ID);   // restores the saved place
    w.show();
    app.run(e => {
        ref place.handle(e);
        match e {
            Message(w, text) => {
                if text == "open" {
                    let files = dialog.openFile({ filters: [{ name: "Notes", extensions: ["md", "txt"] }], multiple: true, parent: w }).or([]);
                    w.eval("opened({json.encode(files)})");
                } else if text.startsWith("link:") {
                    opener.openUrl(text.sub(5, text.len())).or({});
                }
            },
            User(data) => {
                let m = single.decode(data);
                if m != null { w.show(); w.focus(); }
            },
            CloseRequested(w) => w.destroy(),
            _ => {},
        }
    });
}
```

**Shared helpers** (`std.desktop`): `configDir(appId)` is the app's own
settings folder, as Tauri's `app_config_dir` names it:
`$XDG_CONFIG_HOME/<appId>` (else `~/.config/<appId>`) on Linux,
`%APPDATA%\<appId>` on Windows, `~/Library/Application
Support/<appId>` on macOS; it is not created. `validAppId(id)` is what the
services accept as an app id: 1 to 128 ASCII letters, digits, `.`, `-`
and `_`, not starting with `.` or `-` (a bundle identifier).
`tauriOs()` and `tauriArch()` are the names Tauri's updater uses
(`linux`, `windows`, `darwin`; `x86_64`, `aarch64`), `findProgram(name)`
a program's path from `PATH`. The program's executable and working
folder are `std.os`'s `exePath()` and `currentDir()`.

**Dialogs** (`std.desktop.dialog`). `openFile(Options)` returns the
chosen paths (none when cancelled), `openFolder` a folder (several with
`multiple` where the system allows it), `saveFile` the path to save to or
null; the save dialog asks before replacing a file. `Options` has a
`title`, the `directory` to start in, the `fileName` a save dialog
suggests, `filters` (`Filter { name, extensions }`, extensions without
the dot, `"*"` for any file), `multiple` and `parent`, the
`webview.Window` the dialog belongs to. `message(title, text, level,
parent)` shows an OK box; `confirm` OK and Cancel and `ask` Yes and No,
true for OK or Yes (`parent` is optional, last). Every call blocks until
the user answers, so it belongs on the UI thread (a web view's event
handler, or `main` before `run()`), where it is modal to its parent, or
with none to the thread's active window; from a task, wrap it in
`blocking(() => ...)` so a scheduler worker is not held. The parent is
the window's `nativeHandle()`: the dialog's owner HWND on Windows, a
sheet on its `NSWindow` on macOS (`beginSheetModalForWindow:` waited
for in a modal session), and on Linux the portal's `parent_window`
(`x11:<id>`, `portalParent`), zenity's and kdialog's `--attach`, and the
GTK dialog's transient parent; on Wayland only GTK's dialogs get one (the
portal would need the surface exported through xdg-foreign).
On Windows the dialogs are `IFileOpenDialog` and `IFileSaveDialog`
(`FOS_PICKFOLDERS` for folders, `FOS_OVERWRITEPROMPT` for saving), with
the thread joined to a single-threaded COM apartment for the call, and
`MessageBoxW`. On macOS they are `NSOpenPanel`, `NSSavePanel` and
`NSAlert` run modally (on the main thread); filters become
`allowedFileTypes`. On Linux the first choice is the xdg-desktop-portal
`FileChooser` (`OpenFile`, `SaveFile`) on the session bus through
`std.os.dbus`: the request's `handle_token` names its `Request` object,
the `Response` signal brings the answer, and its `file://` URIs become
paths (`uriToPath`). This is the desktop's own chooser, and works inside
Flatpak and Snap. Without a portal (or its FileChooser) it runs zenity,
then kdialog, then GTK 3's `GtkFileChooserNative` when `libgtk-3.so.0`
loads and a display opens. Message boxes on Linux use zenity, kdialog or
a `GtkMessageDialog`, and are printed to stderr when none is there
(`confirm` and `ask` then answer false). `BISMUTH_DIALOG=portal`,
`zenity`, `kdialog` or `gtk` starts the search at that one. The portal's
filters (`portalFilters`, `a(sa(us))`), its options and the helpers'
command lines (`zenityArgs`, `kdialogArgs`, `zenityMessage`,
`kdialogMessage`) are plain functions, checked
by `tests/run/desktop_dialog.bi`. Errors are `DialogError.Unavailable(why)`
(nothing can show a dialog here) and `Failed(why)`.

**Single instance** (`std.desktop.single`). `acquire(appId, Options {
args, timeout })` makes this process the app's primary instance, or
hands `args` (by default `os.args()`, the program first) and the working
folder to the primary that runs and returns `Instance.Secondary`, after
which the caller exits. The primary reads later launches as `Message {
args, cwd }` from `p.messages()`, a `Chan` (a launch waits while 64 are
unread), or `p.forward(proxy)` posts each to a web view's UI thread as
`Event.User`, which `single.decode(data)` turns back into the `Message`
(null for other data); `p.release()` gives the role up early (the
process ending does too). On Linux and macOS the primary holds an
`flock` on `<dir>/<appId>.lock` and listens on the Unix socket
`<dir>/<appId>.sock`, where `<dir>` is `$XDG_RUNTIME_DIR` on Linux or
`$TMPDIR` on macOS, else `/tmp` with the uid in the names; the lock goes
with the process however it ends, so a socket a crashed primary left is
replaced, and a launch that finds the lock held but no socket yet tries
again until `timeout` (5 s). On Windows the primary creates the named
mutex `Local\<appId>` and serves the named pipe
`\\.\pipe\<appId>-<user>` (remote clients refused). A launch sends one
frame, `BSI1`, a little-endian u32 length and the JSON `{"args": [...],
"cwd": "..."}` (at most 1 MiB), and waits for the primary's one-byte
answer. Errors are `SingleError.BadAppId(id)`, `NoAnswer(why)` (the lock
is held but the primary did not take the message in time) and `Io(why)`.

**Window state** (`std.desktop.windowstate`). `save(appId, label,
geometry)` writes a window's place (`webview.Geometry`: the position and
size while not maximized, and whether it is) into
`configDir(appId)/.window-state.json`, and `load(appId, label)` reads it
back, fitted with `fit`: at least 160 by 120, and on Windows no larger
than the virtual screen with the title bar on it; on Linux a negative
position becomes 0. The file is Tauri's: an object per window label with
`width`, `height`, `x`, `y`, `prev_x`, `prev_y`, `maximized`,
`visible`, `decorated` and `fullscreen`, so an app that moves from Tauri
keeps its users' places (for a maximized window `prev_x` and `prev_y`
are where it goes back to). Saving keeps the file's other labels and
members, and writes through a temporary file renamed over it.
`loadFrom(path, label)` and `saveTo(path, label, g)` use another file.
`track(window, appId, label = "main")` restores the saved place and
returns a `Tracker`; `tracker.handle(event)` in the run loop saves the
place when the window is asked to close and, at most every two seconds,
while it moves or resizes. Positions are std.webview's logical pixels,
while Tauri saves physical ones: on a display scaled above 100% a file
Tauri wrote reads back larger by that scale, once.

**Opener** (`std.desktop.opener`). `openUrl(url, schemes)` opens a URL
in the program the user chose for its scheme, the default browser for
http and https; only `http`, `https` and `mailto` unless `schemes` names
others, so a URL from a page or a document never reaches the OS as a
path, a `file:` URL or a scheme that runs something (`javascript:`,
`ms-settings:`, a custom handler). A URL with spaces, quotes or control
characters, or an http(s) URL without a host, is refused as well
(`urlAllowed(url, schemes)` says which; `schemeOf(url)` reads the
scheme). `openPath(path)` opens a file in its default application or a
folder in the file manager; `reveal(path)` shows its folder with it
selected (`SHOpenFolderAndSelectItems`, `open -R`, the
`org.freedesktop.FileManager1` `ShowItems` call, else the folder).
They run `xdg-open` (else `gio open`) on Linux, `open` on macOS and
`ShellExecuteW` on Windows, and return once it has started.
Errors are `OpenError.NotAllowed(url)`, `NotFound(path)` and `Failed(why)`.

**Updater** (`std.desktop.updater`). `check(Config)` (async) asks the
`endpoints` in order for a release newer than `currentVersion`, and
returns an `Update` or null. An endpoint may use `{{current_version}}`,
`{{target}}` (`Config.target`, else the OS), `{{arch}}` and
`{{bundle_type}}`. A static manifest (`latest.json`) has `version`,
`notes`, `pub_date` and `platforms`, keyed `<os>-<arch>-<installer>`
(tried first) or `<os>-<arch>` (`linux-x86_64-appimage`,
`windows-x86_64-nsis`, `darwin-aarch64`), each with `url` and
`signature`; only `Config.target` is tried when it is set. A dynamic
server answers one build's object (`version`, `notes`, `pub_date`, `url`,
`signature`), or 204 when there is nothing newer. An endpoint that fails
or answers another status passes to the next; a release without a build
for this platform is `NoPlatform(keys)`. Versions are SemVer 2.0 with a
leading `v` allowed and missing minor and patch numbers read as 0;
pre-releases come before their release and build metadata is not
compared (`parseVersion`, `compareVersions`). `Update.download(progress)`
fetches the build (with `Config.headers`, up to `maxBytes`), reports
`(bytes so far, total or -1)`, and checks its minisign signature against
`Config.pubkey` before it returns: the key and the signature are taken
in Tauri's form (the base64 of the `.pub` and `.sig` files) or as
minisign writes them. `install(bytes)` checks the signature again before
anything is written, then: on Linux writes the new AppImage beside
`$APPIMAGE`, makes it 0755 and renames it over the old one (a `.tar.gz`
holding one, Tauri v1's format, is unpacked first; an app installed from
a `.deb` or `.rpm` is `Unsupported`); on macOS unpacks the `.app.tar.gz`
beside the running bundle (modes and symbolic links kept, links that
would leave the bundle refused), renames the old bundle aside, moves the
new one into its place and removes the old one; on Windows writes the
NSIS `.exe` or the `.msi` (or a `.zip` holding one) to a folder in
`%TEMP%`, starts it with `ShellExecuteW` and Tauri's arguments
(`installMode` `Passive`: NSIS `/P /UPDATE /R /ARGS <the app's
arguments>`, msiexec `/i <file> /passive /promptrestart
AUTOLAUNCHAPP=True LAUNCHAPPARGS="..."`; `Quiet` and `BasicUi` as
Tauri's), runs `Config.beforeExit` and exits, leaving the installer to
replace the app and start it. `restart()` starts the program again with
its arguments (`$APPIMAGE` in an AppImage) and exits. There is no
administrator prompt on macOS: a bundle the user cannot write is an
`Install` error. Errors are `UpdateError`: `Http(why)`, `Status(code,
url)`, `Manifest(why)`, `Version(text)`, `NoPlatform(keys)`,
`Signature(why)`, `Install(why)` and `Unsupported(why)`.

Tests: `tests/run/desktop_windowstate.bi`, `desktop_updater.bi` (with a
local HTTP server and a key made at run time), `desktop_opener.bi`,
`desktop_dialog.bi` (everywhere), `desktop_single_linux.bi` and
`desktop_single_windows.bi` (a second launch of the test itself). The
dialogs themselves, the installers and the macOS backends have not been
run by CI.

## gamepad

`import std.gamepad;` reads game controllers in one standard layout, named
after an Xbox pad: `Button` (`A` `B` `X` `Y` `Back` `Guide` `Start`
`LeftStick` `RightStick` `LeftShoulder` `RightShoulder` `DpadUp`
`DpadDown` `DpadLeft` `DpadRight`) and `Axis` (`LeftX` `LeftY` `RightX`
`RightY` `LeftTrigger` `RightTrigger`). `A` is the bottom face button on
every pad. Sticks read -1..1 with y positive down, as window coordinates
are; triggers read 0..1. It is a module of its own rather than part of
`std.window`, because pads need no window (a server, a tool, a test), but
it follows the same event model: iterating `Gamepads` reads the devices
once and yields what changed, without blocking.

```bismuth
let pads = gamepad.open()?;
defer pads.close();
for e in pads {
    match e {
        Connected(p) => print("pad {p}: {pads.name(p)}"),
        ButtonDown(p, b) => if b == gamepad.Button.A { pads.rumble(p, 0.3, 0.6, 150.ms); },
        _ => {},
    }
}
let move = pads.leftStick(0);    // vec2, after the deadzone; (0, 0) without pad 0
```

Pads are numbered from 0 as they connect, taking the lowest free number
(up to 8); pads already plugged in arrive as `Connected` events on the
first read. A `Connected` pad's state is taken as it is, so a button held
while plugging in is down without a `ButtonDown`. `AxisMove` carries the
value after deadzones and only comes when that value changes. Programs that
only poll state call `update()` once a frame instead of iterating.

| Platform | Backend | Hotplug | Rumble |
|---|---|---|---|
| Linux | evdev: every `/dev/input/event*` with joystick or gamepad buttons, through the `EVIOCG*` ioctls | inotify on `/dev/input`, else a listing once a second | `FF_RUMBLE`, where the driver has it (xpad, hid-playstation, hid-nintendo, ...) |
| Windows | XInput, loaded at run time (`xinput1_4.dll`, else `xinput9_1_0.dll`); `XInputGetStateEx` for the Guide button | connected slots every read, empty ones once a second | both motors; the duration is kept by the reads |
| macOS | not yet: `open` returns `GamepadError.Unsupported` | | |

**Deadzones.** `Deadzones { left = 0.24, right = 0.27, trigger = 0.12,
outer = 0.98 }` (XInput's recommended thresholds) is set with `open` or
`setDeadzones`. Sticks get a radial deadzone: inside the radius they read
(0, 0), and beyond it the length is rescaled from radius..outer to 0..1
with the direction kept, so the smallest move past the deadzone is small,
and a stick's square corners are clamped to the unit circle. Triggers
get the same on one axis. `gamepad.radial(v, inner, outer)`,
`gamepad.linear(x, inner, outer)` and `applyDeadzones(frame, dz)` are the
functions behind it. `rawState(pad)` gives a frame before deadzones.

**Mappings (Linux).** A pad whose driver follows the kernel's gamepad
layout (`BTN_SOUTH` is A, `BTN_EAST` B, `BTN_X` X, `BTN_Y` Y, `ABS_X`/`ABS_Y`
the left stick, `ABS_RX`/`ABS_RY` the right, `ABS_Z`/`ABS_RZ` the triggers,
`ABS_HAT0X`/`Y` the d-pad), as xpad's Xbox pads do, needs no mapping. Others
are mapped by lines in SDL's [GameControllerDB](https://github.com/mdqinc/SDL_GameControllerDB)
format, found by the pad's GUID (bus, vendor, product and version): first
the program's (`Options.mappings`, `addMappings(text)`, which takes a whole
`gamecontrollerdb.txt`, and the `SDL_GAMECONTROLLERCONFIG` variable), then a
small built-in subset of the database for PlayStation 3, 4 and 5 pads, the
Switch Pro controller, 8BitDo pads, the Logitech Dual Action and the Steam
Deck (`std/gamepad/mappings_db.bi`, under the database's zlib licence,
quoted there). A line with version 0 matches any version, and the name CRC
of newer SDL GUIDs is ignored. Joystick-style pads (`BTN_TRIGGER`,
`BTN_THUMB`, ...) without a line get a guess. `gamepad.parseMapping(line) :
Mapping ! GamepadError` reads one line; `Mapping.str()` writes it back.

**Virtual pads.** `open({ devices: false })` reads no devices (and works on
every OS); `addVirtual(name)` adds a pad that `setVirtual(pad, frame)`
drives, with the same events and deadzones as a device, for replays, tests
and on-screen pads. `Frame` has `down`, `axis`, `leftStick`, `rightStick`,
`press`, `release` and `set`.

**Translators.** The parts that turn raw input into frames are pure code,
compiled on every OS and tested on all of them: `Evdev` (from a
`DeviceInfo` and its `RawEvent`s: `feed` returns true when a `SYN_REPORT`
completes a frame; after `SYN_DROPPED` it sets `resync`) and
`fromXInput(XInputState) : Frame`.

Errors are `GamepadError.Unsupported(why)` (no backend on this OS, or no
XInput DLL), `Failed(why)` (out of pad numbers) and `BadMapping(why)`.
Reading `/dev/input` needs the access logind gives the seat's user to
joysticks, or the `input` group; nodes that can't be read are skipped and
tried again when their permissions change.

## rand

`import std.rand;`, then `var rng = rand.Rng.new(seed);` and call its
methods with `ref`. The generator is xoshiro256** seeded through
splitmix64. Everything in the module uses integer arithmetic and IEEE
`+ - * /` and square roots only: `ln` and `exp` are fdlibm's algorithms
written out in the module rather than the C library's, so a seed gives
the same numbers on Linux, Windows and macOS, on x86-64 and arm64, and at
every optimization level.

| Method | What |
|---|---|
| `below(n) : u64` | uniform in [0, n), with no modulo bias (Lemire's multiply-and-reject; one 64-bit multiply when n fits in 32 bits) |
| `range(lo, hi) : int`, `index(n) : int` | uniform in [lo, hi] (both ends), and in [0, n); `below`, `range` and `index` panic on an empty range |
| `float() : f64`, `float32() : f32`, `uniform(lo, hi) : f64` | uniform in [0, 1) from 53 or 24 random bits, and in [lo, hi) |
| `chance(p)`, `bernoulli(p)` | true with probability p |
| `normal(mean = 0, sd = 1) : f64` | Marsaglia's polar form of Box-Muller; the second value of each pair is kept for the next call |
| `exponential(rate = 1) : f64` | mean 1 / rate |
| `poisson(mean) : int` | Knuth's product method below 10, Hormann's transformed rejection (PTRS) from 10 up |
| `shuffle(ref list)`, `choose(list) : T?` | Fisher-Yates in place; a copy of a random element, null when empty |
| `inCircle(r)`, `onCircle(r) : vec2` | uniform in the disc and on the circle of radius r (rejection, no trigonometry) |
| `inSphere(r)`, `onSphere(r) : vec3` | uniform in the ball and on the sphere (Marsaglia 1972) |
| `inDisc(center, normal, r) : vec3` | uniform on a flat disc facing normal |

`rand.Weighted.new(weights)` builds Vose's alias table in O(n) and returns
`Weighted ! WeightError` (`Empty`, `Invalid(index)` for a negative, NaN
or infinite weight, `ZeroTotal`); `w.sample(ref rng)` then picks index i
with probability `weights[i] / total` in O(1).

## noise

`import std.noise;`, then `let n = noise.Noise.new(seed);`. A `Noise`
holds its seed, Perlin's permutation and OpenSimplex2's gradient tables
(about 1.3 KB), built once; sampling allocates nothing, and methods take
it by `view`. Every function has `(x, y)`, `(x, y, z)`, `vec2` and `vec3`
forms, takes and returns `f32`, and uses no libm, so a seed gives the same
values on every platform.

| Method | What |
|---|---|
| `perlin` | Ken Perlin's improved gradient noise (2002), about [-1, 1]; repeats every 256 units |
| `simplex` | OpenSimplex2 (KdotJPG's patent-free simplex-style noise), about [-1, 1], no repeat; 3D uses the lattice's general orientation |
| `simplexXY(x, y, z)` | 3D OpenSimplex2 oriented for good XY slices: 2D maps that change along z (height or time) |
| `value` | random values at integer points, blended with Perlin's quintic, in [-1, 1] |
| `cellular(.., metric = .Euclidean, jitter = 1.0) : Cell` | Worley noise: `f1` and `f2`, the distances to the nearest and second nearest feature point, and `id`, a hash of the nearest point's cell (`value()` maps it to [0, 1)). `Metric` is `Euclidean`, `EuclideanSq`, `Manhattan` or `Chebyshev`; jitter (0 to 1) is how far points stray from their cell's center. F1 and F2 are exact: after the 3x3 (3x3x3) block, rings further out are searched while they could hold a nearer point |
| `fbm(.., Fractal)` | fractal Brownian motion, about [-1, 1] |
| `ridged(.., Fractal)` | ridged multifractal, in [0, 1] |
| `turbulence(.., Fractal)` | the sum of absolute octaves, in [0, 1] |

`Fractal { basis = .Simplex, octaves = 5, lacunarity = 2.0, gain = 0.5 }`
picks the basis (`Perlin`, `Simplex` or `Value`); each octave gets its own
seed (or offset, for Perlin), so octaves don't line up at the origin. On
an x86-64 desktop one sample takes about 11 ns for 2D Perlin, 16 ns for
2D OpenSimplex2, 25 ns for 3D OpenSimplex2 and 80 ns for 2D cellular.

## collision

`import std.collision;`. Shapes are plain structs on `vec2` and `vec3`,
passed by copy; overlap tests count touching as overlapping.

| Type | Methods |
|---|---|
| `Aabb2 { min, max }`, `Aabb3` | `new` `around(center, half)` `of(points)` `center` `half` `size` `area`/`perimeter` (2D), `volume`/`surfaceArea` (3D) `contains(point or box)` `overlaps(box, Circle or Sphere)` `union` `grown(m)` `closestPoint` |
| `Circle { center, radius }`, `Sphere` | `contains` `overlaps(Circle/Sphere, box, capsule)` `bounds` `closestPoint` |
| `Capsule2 { a, b, radius }`, `Capsule3` | `overlaps(capsule, Circle/Sphere)` `contains` `bounds` `segment` |
| `Segment2 { a, b }`, `Segment3` | `closestPoint(p)`; `closestPoints(other) : (p, q)` (Ericson 5.1.9); `Segment2.intersect(other) : vec2?` (collinear overlaps meet at the point nearest `a`) |
| `Triangle { a, b, c }` | `normal` `bounds` `closestPoint` (by Voronoi region) |
| `Plane { normal, d }` | `new(normal, point)` `through(a, b, c)` `distance` (signed) `closestPoint` `overlaps(Sphere or Aabb3)` |
| `Ray2 { origin, dir }`, `Ray3` | `at(t)`; `cast(shape, maxT) : Hit2? / Hit3?` against `Aabb2` (slab), `Circle`, `Segment2`; `Aabb3` (slab), `Sphere`, `Plane`, `Triangle` (Moller-Trumbore, with barycentric `u` `v`) |

`dir` need not be unit length: `t` is in units of it. Casts report hits
with `0 <= t <= maxT`, with a normal facing the ray; a ray starting inside
a solid shape hits at `t = 0` with a zero normal. Planes and triangles are
hit from either side.

Polygons are `List<vec2>` in either winding. `pointInPolygon(poly, p)`
uses the even-odd rule (any simple polygon). `sat(a, b) : Contact2?` tests
two convex polygons on every edge normal: null when they are apart,
otherwise the `normal` (unit, from a toward b) and `depth` of least
penetration, with containment handled; `contact.mtv()` is the translation
that moves `a` out. `closestPointOnPolygon(poly, p)` finds the nearest
point on the boundary.

Two broadphases, generic over `Aabb2` and `Aabb3`, hand out int ids,
reuse removed ids and their storage, and append to a caller's list in
`query`, so a steady frame allocates nothing:

- `Grid<B>.new(cellSize)`: `insert(box) : int` `remove(id)` `move(id, box)`
  (refiles only when the box's cells change) `query(box, ref out)` (each
  overlapping id once, exact box test) `box(id)` `len`. For many objects of
  similar size.
- `AabbTree<B>.new(margin = 0.1)`: `insert(box, data) : int` `remove(id)`
  `move(id, box) : bool` (false while the box stays inside its fat box)
  `query(box, ref out)` (ids whose fat boxes overlap) `data(id)` `fatBox(id)`
  `height` `len` `validate`. A Box2D-style dynamic tree: leaves hold boxes
  grown by the margin, insertion picks the sibling of least added cost
  (perimeter in 2D, surface area in 3D) and rotations keep it balanced.

## deflate, zlib and gzip

`std.io.compression.deflate` is the deflate format (RFC 1951) and its two
framings: zlib (RFC 1950: a 2-byte header and an Adler-32 trailer, as in
PNG) and gzip (RFC 1952: a member header and a CRC-32 and size trailer, as
in `.gz` files and HTTP). `std.io.compression.zlib` and
`std.io.compression.gzip` are the one-shot calls for each framing.

```bismuth
import std.io.compression.gzip;
import std.io.compression.deflate;

let gz = gzip.compress(data);              // level 6; 0 (stored) to 9
let back = gzip.decompress(gz)?;           // List<u8> ! DecodeError
let raw = deflate.compress(data, 1);       // bare deflate, fastest level
let crc = deflate.crc32(data);
```

- **Levels** follow zlib's: 0 stores, 1-3 match greedily, 4-9 match lazily
  with longer hash chains; outside 0..9 is clamped. Each block of up to
  16384 symbols is written stored, with the fixed codes or with its own
  codes, whichever is smallest. Output sizes are within a few bytes of
  zlib's at the same level.
- **Errors** are `DecodeError.TooShort` (the input ends inside the stream)
  and `BadData` (anything else: a bad code, a distance too far back, a
  wrong checksum or size, bytes after the end). gzip files may hold
  several members, decoded in turn; other bytes after the end are an
  error. zlib streams with a preset dictionary are refused.
- **Bounded output**: `decompressMax(slice, limit, format = Raw, sizeHint = -1)`
  is `decompress` that gives null, without decoding further, as soon as the
  output would pass `limit` bytes, so an input that expands enormously (a
  "zip bomb") costs at most `limit`; `sizeHint` (a size a container lists)
  sizes the output at once when the input could plausibly reach it.
  `gzip.decompressMax(bytes, limit)` is the same for `.gz` data.
- **Gzip output** has no name and time 0, and says OS "unknown", so the
  same input gives the same bytes on every OS.
- **Streams**: `Inflater.new(format)` takes input a chunk at a time
  (`write(chunk)` returns the bytes decoded so far), keeping the 32 KiB
  window and any partial input; `finish()` is `TooShort` unless the stream
  ended. Bytes after the end are `BadData` (except another gzip member);
  `Inflater.new(format, true)` stops at the end instead and keeps them for
  `rest()`, as RFC 7692's trailer after a final block needs. `Deflater.new(format, level)` returns compressed bytes from
  `write` as blocks fill and the rest, with the trailer, from `finish()`.
  Its output is the one-shot output for the same input whatever the chunks.
- **Speed**: decoding uses table lookups (10 bits for literals and
  lengths, 8 for distances, with subtables for longer codes) and a fast
  loop that refills 64 bits at once and copies matches 8 bytes at a time;
  it runs at or above zlib's speed. Matching runs 1.1 to 1.6 times zlib's
  time at levels 6-9. CRC-32 folds runs of 64 bytes or more by carry-less
  multiplication where the CPU has it (PCLMULQDQ on x86-64, PMULL on
  arm64: about 30 GB/s on long data here, against slicing-by-8's 1.8),
  the rest by slicing-by-8 (`useCrcFold(false)` keeps it to the tables,
  for tests and benchmarks); Adler-32 sums in 64 bits.

## zip and tar archives

`std.io.archive.zip` and `std.io.archive.tar` read and write the two
archive formats on top of `std.io.compression.deflate`. Both read from
bytes held in memory (`fs.read` the file first) and build archives in
memory. `std.io.archive` has what they share: `cleanPath`, which callers
unpacking an archive use to refuse path traversal.

```bismuth
import std.io.fs;
import std.io.archive;
import std.io.archive.zip;
import std.io.archive.tar;

enum UnpackError { from zip.ZipError, from tar.TarError, from archive.PathError, from fs.IoError, Link(str name), TooBig }

// unpack unpacks a zip or a .tar.gz under dest: no links, 128 MiB a file,
// 512 MiB in all
fn unpack(List<u8> data, str dest) : void ! UnpackError {
    if zip.isZip(data) {
        let z = zip.Reader.open(data)?;
        var total = 0;
        for i, e in z.entries {
            total += e.size;
            if total > 512.mb { return UnpackError.TooBig; }
            if e.isDir() { continue; }
            if !e.isFile() { return UnpackError.Link(e.name); }
            save(dest, archive.cleanPath(e.name)?, z.read(i, 128.mb)?)?;
        }
    } else {
        let t = tar.Reader.openGz(data, 512.mb)?;
        for i, e in t.entries {
            if e.isDir() { continue; }
            if !e.isFile() { return UnpackError.Link(e.name); }
            save(dest, archive.cleanPath(e.name)?, t.read(i))?;
        }
    }
}

fn save(str dest, str rel, view List<u8> bytes) : void ! fs.IoError {
    let path = "{dest}/{rel}";
    fs.makeDirAll(path.sub(0, path.rfind("/") ?? 0))?;
    fs.writeBytes(path, bytes)?;
}

var w = zip.Writer.new();
ref w.add("readme.txt", text.bytes());          // deflated, or stored when that is smaller
ref w.add("photo.jpg", jpg, zip.Method.Stored);
ref w.addDir("empty");
fs.writeBytes("out.zip", ref w.finish())?;
```

**zip** (`std.io.archive.zip`):

- `Reader.open(bytes) : Reader ! ZipError` finds the end record (searching
  back over a comment), follows the zip64 locator when there is one, and
  reads the whole central directory into `entries` (a `List<Entry>`), so
  listing costs nothing more. The `List` passed is taken over without a
  copy when it is the caller's last use of it. Bytes before the archive (a
  self-extractor's stub) are allowed: offsets are shifted by them, as
  Info-ZIP does.
- `Entry` has `name` (as stored: `/` between parts; a directory's ends in
  `/`), `kind` (`Kind.File`, `Dir` or `Symlink`, from the name and the
  attributes), `method` (`Method.Stored`, `Deflate` or `Other(code)`),
  `size`, `compressedSize`, `crc`, `mode` (Unix permission bits from the
  external attributes of an archive made on Unix, else 0), `mtime` (Unix
  seconds: the extended timestamp or NTFS time when present, else the DOS
  time read as UTC), `encrypted` and `comment`; `isDir()`, `isFile()`,
  `isSymlink()`. Names flagged UTF-8 must be UTF-8; others are read as
  UTF-8 when they are valid UTF-8 (as Info-ZIP and most tools write them)
  and as code page 437 otherwise; an Info-ZIP Unicode path field wins
  while it matches.
- `read(i, maxBytes = int.max) : List<u8> ! ZipError` decodes one entry.
  It is `TooLarge` when the listed size is over `maxBytes`, and it never
  decodes past the listed size, so a zip bomb costs at most what `entries`
  says (check the sizes, and their sum, before reading). Data that decodes
  to another size is `BadData`, a CRC-32 that differs `BadCrc`. `find(name)
  : int?` and `readName(name, maxBytes)` look an entry up by its exact
  name.
- `ZipError` is `NotZip`, `Truncated`, `Malformed(what)` (a record or field
  that does not hold together), `Unsupported(what)` (several disks,
  encryption, a method other than stored and deflate), `BadData(name)`,
  `BadCrc(name)`, `TooLarge(name)` or `NotFound(name)`. Malformed input is
  an error, never a panic.
- `Writer.new(level = 6)`: `add(name, bytes, method = Method.Deflate, mode =
  0o644)` (deflate falls back to stored when it would not shrink the data),
  `addDir(name, mode = 0o755)` (adds the `/`), `addSymlink(name, target)`,
  then `finish() : List<u8>`. The public fields `level` and `mtime` (Unix
  seconds, default 1980-01-01) apply to the entries added next; times go in
  the DOS fields and an extended timestamp. Entries are written Unix-style
  (mode in the external attributes), names flagged UTF-8 when not ASCII,
  with zip64 fields for sizes and offsets past 4 GiB and end records past
  65534 entries. `dosToUnix` and `unixToDos` convert DOS times.

**tar** (`std.io.archive.tar`):

- `Reader.open(bytes) : Reader ! TarError` reads a plain tar;
  `Reader.openGz(bytes, maxBytes = int.max)` a `.tar.gz` (or a plain tar
  when the bytes are not gzip), `TooLarge` without decoding further when
  the tar inside is over `maxBytes`. Every header is checked when it
  opens (its checksum, signed or unsigned, and its numbers: octal or GNU
  base-256), and `entries` lists them in order. The archive ends at a zero
  block or at the end of the data on a block boundary.
- POSIX ustar (with its name prefix), pax (`x` records for the next entry
  and `g` for all that follow: `path`, `linkpath`, `size`, `mtime`, `uid`,
  `gid`, `uname`, `gname`) and GNU long names and link targets (`L`, `K`)
  are folded into the `Entry` they describe: `name`, `kind`, `size`,
  `mode`, `mtime`, `linkName`, `uid`, `gid`, `uname`, `gname`; `isDir()`,
  `isFile()`, `isLink()`.
- `Kind` is `File` (`0`, NUL, `7`, or an old-style name ending in `/` is
  `Dir`), `Dir`, `Symlink`, `Hardlink`, `CharDevice`, `BlockDevice`,
  `Fifo` or `Other(flag)` (GNU sparse files, volume labels, ...). Links and
  devices are only reported, never followed or created; refuse them when
  unpacking untrusted archives. Their size is 0.
- `read(i) : List<u8>` copies an entry's data; `find(name)` gives the last
  entry of that name (the one tar leaves on disk), `readName(name)`.
- `TarError` is `Truncated`, `BadHeader(what)`, `TooLarge`,
  `Gzip(DecodeError)` or `NotFound(name)`.
- `Writer.new()`: `add(name, bytes, mode = 0o644)`, `addDir(name, mode =
  0o755)`, `addSymlink(name, target)`, `addHardlink(name, target)`, then
  `finish()` or `finishGz(level = 6)`. The public fields `mtime`, `uid`,
  `gid`, `uname` and `gname` apply to the entries added next. Headers are
  ustar; a name over 100 bytes is split into the prefix field when it can
  be, and otherwise, like a link target over 100 bytes or a size, time or
  id past the octal fields, goes in a pax record.

**Paths** (`std.io.archive`): `cleanPath(name) : str ! PathError` turns an
entry name into a relative path: `\` counts as `/`, empty and `.` parts
and a trailing `/` are dropped (`"./a//b/"` is `"a/b"`). It refuses `Empty`
names (nothing left), `Absolute` ones (`/x`, `\x`, `//host/share`, `C:x`),
`Parent` (a `..` part, or one of dots and spaces Windows could read as
`..`) and `BadChar` (control characters, NUL, `:`). Joined to a folder,
the result stays inside it. `decodeName(bytes, cp437 = false)` is the name
decoding both readers use.

## PNG

`std.gfx.image` reads and writes PNG.

```bismuth
import std.gfx.image;

let img = image.Image.loadPng("sprite.png")?;    // Image ! PngError
let c = img[3, 4];                                // Color { r, g, b, a }
img.savePng("copy.png")?;                         // void ! fs.IoError
let bytes = img.encodePng(image.PngColor.Rgba, 9);
```

Decoding takes every colour type (grey, RGB, palette, grey with alpha,
RGBA), bit depths 1, 2, 4, 8 and 16, `tRNS` transparency, Adam7
interlacing and all five filters, and checks every chunk's CRC. Pixels
come out as 8-bit RGBA: low depths are scaled up, 16-bit samples keep their
high byte. Other ancillary chunks (gamma, colour profiles, text) are
skipped, so colours are not gamma-corrected; an unknown critical chunk is
an error. `PngError` is `NotPng`, `BadChunk(what)`, `BadCrc(chunk)`,
`BadHeader`, `BadData`, `Unsupported(chunk)`, `TooLarge` (over 2^28
pixels) or an `fs.IoError` from `loadPng`.

Encoding writes 8-bit RGB, or RGBA when a pixel is not opaque
(`PngColor.Auto`, the default; `Rgb` and `Rgba` force one), with one IDAT
chunk. Each row takes the filter whose output has the smallest sum of
absolute values, libpng's heuristic. `rgba()` and `Image.fromRgba(w, h,
bytes)` move pixels in and out as bytes.

## model

`std.gfx.model` reads 3D models into one scene structure and runs
post-processing steps over them, covering what Assimp does for FBX, OBJ
and STL, and writes OBJ and STL.

```bismuth
import std.gfx.model;

let scene = model.load("ship.fbx", model.Options {
    steps: model.Steps.realtimeQuality(),    // triangles, smooth normals, tangents, shared vertices, ...
})?;
for m in scene.meshes {
    // positions, normals, tangents, bitangents, uvs[k], colors[k]; indices, three per triangle
    let mat = scene.materials[m.material];
    print("{m.name}: {m.vertexCount()} vertices, {m.faceCount()} triangles, {mat.name}");
}
let world = scene.globalTransform(scene.findNode("turret") ?? 0);
let pose = scene.pose(0, 12.5);                 // animation 0 at tick 12.5: every node's matrix
model.save(scene, "ship.obj")?;                 // and ship.mtl
```

**Formats.** The format comes from `Options.format`, the file's bytes or
its extension:

- **FBX** 7.1 to 7.7 (FBX 2011 and later), binary (32- and 64-bit record
  headers, zlib-compressed arrays) and ASCII. Models become nodes with the
  full FBX transform (translation, rotation in any of the six orders,
  scaling, pre- and post-rotation, rotation and scaling pivots and
  offsets); geometric transforms are baked into the meshes. Meshes are
  split by material, with normals, tangents and binormals, every uv and
  colour layer, mapped by polygon vertex, control point, polygon or all
  the same, direct or indexed; line geometry becomes line meshes. Skins
  give bones (offset `TransformLink⁻¹ · Transform`) and weights; blend
  shapes give morph targets. Materials read Lambert and Phong properties
  and Maya and 3ds Max PBR ones, keep every property in `extra`, and get
  their textures (and layered textures) by the property they feed, with
  uv transforms, wrap modes and uv set names; video content becomes
  embedded textures (`path` is `"*N"`). Cameras, lights and every
  animation stack follow: animated nodes are sampled at each key time
  through the whole transform, so pivots and pre-rotations animate
  right, and blend shape channels become morph animations.
  `GlobalSettings` go to `scene.meta` (`UpAxis`, `UnitScaleFactor`,
  `FrameRate`, ...). Older files are `Unsupported`, as in Assimp.
- **OBJ** with its **MTL** libraries: `o` and `g` become nodes and each
  material used a mesh; polygons, polylines (`l`) and points (`p`),
  negative indices, `v x y z r g b` vertex colours, 3-component texture
  coordinates. MTL colours, `Ns` `Ni` `d` `Tr` `illum`, the PBR extension
  (`Pr` `Pm` `Ps` `Pc` `Pcr` `aniso` `anisor`) and every map with its
  options (`-o` `-s` `-clamp` `-bm` `-type sphere`, ...). Free-form
  curves and surfaces are skipped with a warning.
- **STL**, ASCII (every `solid` a mesh) and binary, with per-facet
  colours as VisCAM, SolidView and Materialise Magics write them.

**The scene** is Assimp's `aiScene` as Bismuth data: `nodes[0]` is the
root, and nodes, meshes, materials, embedded textures, animations,
cameras and lights refer to each other by index or name. A `Mesh` has
`positions` and, empty or one per vertex, `normals`, `tangents`,
`bitangents`, up to 8 `colors` and 8 `uvs` sets (`uvComponents`,
`uvNames`); faces are `indices` back to back, `faceSizes[i]` each, with
`primitives` the `PRIM_POINT`/`LINE`/`TRIANGLE`/`POLYGON` bits present;
`bones` (`offset`, `weights`, `node`, `armature`), `morphTargets` and
`bounds`. Loaders give one vertex per face corner; `joinIdenticalVertices`
shares them. A `Material` has the classic colours and factors, the PBR
ones (`baseColor`, `metallic`, `roughness`, sheen, clearcoat,
transmission, anisotropy), `shading`, `twoSided`, `textures`
(`TextureRef { kind, path, uvSet, mapping, op, wrapU, wrapV, transform,
strength }`, `kind` one of Assimp's texture types) and `extra`.
`Animation`s hold node channels (translation, rotation and scaling keys,
with `sample`) and morph channels, timed in ticks. `Scene.pose(anim, t)`
and `Scene.skin(mesh, node, pose)` give the matrices to draw a skinned,
animated mesh.

**Post-processing** runs the steps set in `Steps`, in Assimp's order, from
`load` or later with `process(ref scene, steps)`. Most are also a function
of their own (`model.triangulate(ref scene)`) that takes the step's
settings as arguments (`model.optimizeGraph(ref scene, keepNodes)`,
`model.removeRedundantMaterials(ref scene, keepMaterials)`,
`model.genSmoothNormals(ref scene, smoothAngle)`); `splitLargeMeshes` is
two, `splitLargeMeshesByFaces(ref scene, limit)` and
`splitLargeMeshesByVertices(ref scene, limit)`, `forceGenNormals` is
`genNormals(ref scene, true)`, and `dropNormals` and `genBoundingBoxes`
are steps only:

| Step | Does |
|---|---|
| `triangulate` | polygons to triangles by ear clipping, so concave ones come out right |
| `genNormals`, `genSmoothNormals` (`smoothAngle`), `forceGenNormals`, `dropNormals` | flat or smooth normals where a mesh has none (or always) |
| `calcTangentSpace` (`uvSetForTangents`, `tangentSmoothAngle`) | tangents and bitangents |
| `joinIdenticalVertices` | one vertex per distinct set of attributes, weights and morph data |
| `makeLeftHanded`, `flipUvs`, `flipWindingOrder` | Direct3D conventions; together `Steps.convertToLeftHanded()` |
| `preTransformVertices` (`preTransformNormalize`) | every node's transform baked in, one node, meshes merged by material |
| `optimizeGraph` (`keepNodes`), `optimizeMeshes`, `findInstances` | fewer nodes and draw calls, shared duplicate meshes |
| `removeRedundantMaterials` (`keepMaterials`), `removeComponent` (`remove`) | drop duplicate and unused materials, or whole kinds of data |
| `sortByPrimitiveType` (`sortRemove`), `findDegenerates` (`degeneratesRemove`, `degeneratesCheckArea`), `findInvalidData` | one primitive kind per mesh; collapsed faces; NaN, constant or zero data and repeated keys |
| `splitLargeMeshes` (`splitVertexLimit`, `splitTriangleLimit`), `splitByBoneCount` (`maxBones`) | meshes that fit index and bone palette limits |
| `limitBoneWeights` (`maxWeights`), `debone` (`deboneThreshold`, `deboneAllOrNone`), `populateArmatureData` | four weights per vertex; rigid parts out of the skin; bone nodes and armatures |
| `improveCacheLocality` (`cacheSize`) | triangle order for the vertex cache (Tipsify), vertices in first-use order |
| `genUvCoords`, `transformUvCoords` | sphere, cylinder, box and plane mappings and uv transforms baked into uv sets |
| `fixInfacingNormals`, `globalScale` (`scale`), `embedTextures`, `genBoundingBoxes`, `validateDataStructure` | flipped normals turned out; scale; texture files read into the scene; bounds (always kept); a broken scene is `ModelError.Invalid` |

`Steps.realtimeFast()`, `realtimeQuality()` and `realtimeMaxQuality()`
are Assimp's `aiProcessPreset_TargetRealtime_*`. Importer settings are in
`Options.obj`, `Options.fbx` (Assimp's `AI_CONFIG_IMPORT_FBX_*`: which
layers, materials, textures, cameras, lights, animations, skins and blend
shapes to read, strict mode, conversion to metres) and `Options.stl`.
Problems a file survives are in `scene.warnings`; the rest are
`ModelError` (`UnknownFormat`, `Syntax(format, line, msg)`,
`BadData(format, msg)`, `Unsupported(format, what)`, `Invalid(msg)`, or an
`fs.IoError`), with `message()`.

**Writing.** `save(scene, path)` writes `.obj` (and the `.mtl` beside it)
or binary `.stl`; `encodeObj(scene, mtlName) : (str, str)`,
`encodeStl(scene) : List<u8>` and `encodeStlAscii(scene)` give the bytes.
Every mesh instance is written with its node's global transform, as
Assimp's exporters do.

## crypto

`import std.crypto;` gives hashes, MACs, key derivation, authenticated
encryption, key agreement, signatures, key encodings and secure random
numbers, all written in Bismuth like the rest of std: no OpenSSL or
other library is loaded. Code that handles secrets (ciphers, MACs,
private keys, nonces, shared secrets) runs in constant time: no branch
or memory index depends on a secret, AES included (it is bitsliced, with
no tables).

```bismuth
import std.crypto;

let digest = crypto.sha256("abc".bytes());             // List<u8>, 32 bytes
print("{digest:hex}");                                 // ba7816bf...
var h = crypto.Hasher.new(crypto.Hash.Sha512);         // incremental
ref h.update(part1);
ref h.updateStr("more");
let d = h.finish();                                    // h can go on
let tag = crypto.hmac(crypto.Hash.Sha256, key, message);
let ok = crypto.constantTimeEqual(tag, received);
let okm = crypto.hkdf(crypto.Hash.Sha256, secret, salt, info, 32)?;
let stored = crypto.pbkdf2(crypto.Hash.Sha256, password, salt, 600000, 32)?;
let salt = crypto.randomBytes(16)?;

let aead = crypto.ChaCha20Poly1305.new(key)?;          // or AesGcm.new(key)
let sealed = aead.seal(nonce, plaintext, aad)?;        // ciphertext and 16-byte tag
let plain = aead.open(nonce, sealed, aad)?;            // CryptoError.Auth if tampered

let me = crypto.X25519Key.generate()?;
let shared = me.sharedSecret(peerPublicKey)?;          // 32 bytes; feed it to HKDF

let signer = crypto.Ed25519Key.generate()?;
let sig = signer.sign(message);                        // 64 bytes
let valid = crypto.ed25519Verify(signer.publicKey(), message, sig);

let key = crypto.PrivateKey.parsePem(pemText)?;        // PKCS#8 or SEC1
let pub = crypto.PublicKey.parsePem(pubText)?;         // SubjectPublicKeyInfo or PKCS#1
```

| Item | What |
|---|---|
| `Hash` | `Sha1` `Sha224` `Sha256` `Sha384` `Sha512` `Sha512t256` (SHA-512/256); `size()` (digest bytes), `blockSize()`, `name()` |
| `Hasher` | `new(Hash)`, `update(List<u8>)`, `updateStr(str)`, `updateSlice(Slice<u8>)`, `finish() : List<u8>` (works on a copy, so hashing can go on), `reset()`, `algo()`; `updateRaw(p, n)` and `finishInto(p)` for pointers, in `unsafe` |
| One-shot | `hash(Hash, bytes)`, `sha1` `sha224` `sha256` `sha384` `sha512` `sha512t256` |
| BLAKE2b (RFC 7693) | `Blake2b.new(size = 64)`, `Blake2b.newKeyed(key, size = 64)` (a MAC; keys up to 64 bytes) with `update` `updateStr` `updateSlice` `updateRaw` `finish` `finishInto` `reset` `size`; `blake2b(data, size = 64)` (digests of 1 to 64 bytes; another size panics), `blake2bKeyed(key, data, size = 64)` |
| `Hmac` | `new(Hash, key)`, `update`, `updateStr`, `finish`, `algo`; `hmac(Hash, key, data)` (RFC 2104; keys longer than a block are hashed first) |
| HKDF (RFC 5869) | `hkdf(Hash, secret, salt, info, length)`, `hkdfExtract(Hash, secret, salt)`, `hkdfExpand(Hash, prk, info, length)`; `length` up to 255 hash lengths |
| PBKDF2 (RFC 8018) | `pbkdf2(Hash, password, salt, iterations, length)` |
| Compare | `constantTimeEqual(a, b)`: whether two byte lists are equal, in time that depends only on their length |
| Random | `randomBytes(n)`, `randomFill(ref buf)`, `randomU64()`, `randomRange(lo, hi)` (unbiased): the OS's generator, `getrandom` on Linux, `getentropy` on macOS, `ProcessPrng` (bcryptprimitives.dll) on Windows |

### Authenticated encryption

`ChaCha20Poly1305` (RFC 8439), `XChaCha20Poly1305` (24-byte nonces,
draft-irtf-cfrg-xchacha) and `AesGcm` (NIST SP 800-38D; 16, 24 or
32-byte keys) share one shape: `new(key)`, then `seal(nonce, plaintext,
aad) : List<u8> ! CryptoError` returns the ciphertext followed by a
16-byte tag, and `open(nonce, sealed, aad)` checks the tag in constant
time before decrypting, returning `CryptoError.Auth` on any mismatch.
`sealRaw` and `openRaw` do the same on pointers, in place if `dst` is
`src` (`openRaw` returns `bool`; `AesGcm`'s take the nonce's length
too). `ChaCha20Poly1305` and `XChaCha20Poly1305` also have
`sealInPlace(nonce, ref buf, aad)`, which encrypts a `List<u8>` where it
is and appends the tag, and `openInPlace(nonce, ref buf, aad)`, which
checks and decrypts it and drops the tag (a wrong tag leaves `buf` as it
was): nothing is allocated once `buf` has room for the tag. On x86-64
ChaCha20 runs eight blocks at a time in AVX2 registers (std.core's
`u32x8`) where the CPU has them (CPUID, asked once), else four in SSE2
registers (`u32x4`), rotating by 8 and 16 bits with SSSE3's `pshufb`
where the CPU has it; on arm64 it runs four blocks at a time in NEON
registers. The AEAD runs Poly1305 in the gaps of ChaCha20's rounds;
Poly1305 works in 64-bit limbs on every target, except that on x86-64 with
AVX2 runs of 36 blocks (576 bytes) or more go four blocks at a time in
26-bit limbs (`vpmuludq` on `u64x4`s, with r^4, as OpenSSL's and
RustCrypto's AVX2 code), about twice as fast on long messages.
`useAvx2(false)` keeps ChaCha20 to the four-block code and Poly1305 to
its 64-bit limbs, and `useSsse3(false)` the four-block code to SSE2's
shifts (for tests and benchmarks; `true` turns them back on where the
CPU has them, and each returns whether its code is on: `useAvx2`
ChaCha20's).
`AesGcm` and `Aes` run on the CPU's AES and carry-less multiply
instructions where it has them (asked once): AES-NI and PCLMULQDQ on
x86-64 (CPUID leaf 1), the AES and PMULL instructions on arm64
(`getauxval(AT_HWCAP)` on Linux, always on macOS). Counter mode runs
eight blocks at a time, GHASH four blocks per reduction (Karatsuba, with
H to H^4 kept with the key); elsewhere both are the bitsliced,
table-free software. `useAesInstructions(false)` keeps them to the
software (`true` turns the instructions back on; it returns whether
they are on), and `aesInstructions()` says whether they are on. Sealing
and opening 1400-byte packets on an x86-64 Xeon: AES-128-GCM about 3.2
GB/s with them against 0.11 without, ChaCha20-Poly1305 1.6.
`nonceSize()` and `tagSize()` say 12 (24) and 16.

A nonce must never repeat under one key. 12-byte nonces suit a counter
(TLS 1.3 and QUIC derive them from the record number); for random nonces
use `XChaCha20Poly1305`. `AesGcm` takes nonces of any length, as the
standard allows, but 12 bytes is the normal and faster case.

| Item | What |
|---|---|
| `ChaCha20` | the stream cipher: `new(key, nonce, counter = 0)` (12-byte nonce, or 24 for XChaCha20), `xor(data) : List<u8>`, `xorInPlace(ref data)`, `xorRaw(src, dst, n)`; each call continues the keystream; running the 32-bit block counter out panics instead of repeating it |
| One-shot | `chacha20(key, nonce, counter, data)` (encrypts and decrypts), `hchacha20(key, nonce16)` |
| `Poly1305` | the one-time MAC: `new(key32)`, `update`, `updateRaw`, `finish`; `poly1305(key, data)` |
| `Aes` | `new(key)` (AES-128, -192, -256), `encryptBlock(block16)`, `encrypt4(src, dst)` (four blocks, in `unsafe`): the forward cipher only, as GCM, CTR and QUIC header protection use it; on the CPU's AES instructions where it has them |

AES is bitsliced four blocks at a time (BearSSL's `aes_ct64` design, the
S-box as Boyar and Peralta's circuit) and GHASH multiplies through
integer multiplications of bit-sparse words (`ghash_ctmul64`), so neither
has tables to leak through the cache. Without the CPU's AES instructions
this is several times slower than OpenSSL: about 130 MB/s for AES-CTR and
70 MB/s for AES-GCM on a recent x86-64; ChaCha20-Poly1305 runs at about
1.3 GB/s for 1400-byte packets with AVX2 (0.75 GB/s with SSE2) and is the
better choice where both peers allow it.

### Key agreement and signatures

| Item | What |
|---|---|
| X25519 (RFC 7748) | `X25519Key.generate()`, `fromBytes(32)`, `bytes()`, `publicKey()` (32 bytes), `sharedSecret(peer)`; `x25519(scalar, point)`, `x25519Base(scalar)`. An all-zero result (a small-order peer point) is `CryptoError.Invalid` |
| Ed25519 (RFC 8032) | `Ed25519Key.generate()`, `fromSeed(32)`, `seed()`, `publicKey()`, `sign(message)` (64 bytes), `signWithContext(message, context)` (Ed25519ctx), `signPrehashed(message, context)` (Ed25519ph); `ed25519Verify(pub, message, sig)`, `ed25519VerifyWithContext`, `ed25519VerifyPrehashed`, `ed25519PublicKey(seed)` |
| P-256 (FIPS 186-5) | `P256Key.generate()`, `fromBytes(32)` (big-endian scalar in 1..n-1), `bytes()`, `publicKey()`, `sign(message)` and `signDigest(digest)` (ECDSA with SHA-256, nonces from RFC 6979, `r ‖ s` as 64 bytes), `signDer`, `sharedSecret(peer)` (ECDH, the 32-byte x-coordinate), `encodeSec1()`, `parseSec1(der)`; `P256PublicKey.decode(sec1)` (65-byte uncompressed or 33-byte compressed, checked to be on the curve), `encode()`, `encodeCompressed()`, `verify(message, sig)`, `verifyDigest`, `verifyDer`; `ecdsaSignatureToDer(raw)`, `ecdsaSignatureFromDer(der)` |
| P-384 (verification) | `P384PublicKey.decode(sec1)` (97 or 49 bytes), `encode()`, `verify(Hash, message, sig)` (`r ‖ s`, 96 bytes), `verifyDigest`, `verifyDer(Hash, message, der)`: for the CA certificates that sign with P-384 keys. Public data only, so not constant-time; about 3 ms per signature |
| RSA (RFC 8017) | `RsaPublicKey.new(modulus, exponent)`, `parsePkcs1(der)`, `encodePkcs1()`, `encodePem()`, `bits()`, `size()`, `modulus()`, `exponent()`; `verifyPkcs1v15(Hash, message, sig)`, `verifyPkcs1v15Digest`, `verifyPss(Hash, message, sig, saltLen = PSS_SALT_HASH)` (or a length, or `PSS_SALT_ANY`), `verifyPssDigest`; 1024 to 16384-bit keys. `RsaPrivateKey.parsePkcs1(der)`, `fromParts(parts)`, `encodePkcs1()`, `encodePem()`, `publicKey()`, `signPkcs1v15(Hash, message)`, `signPkcs1v15Digest`, `signPss(Hash, message)` (a random salt as long as the digest), `signPssDigest`. No key generation |

Ed25519 verification is strict: `S` must be below the group order, both
points must be canonical encodings, public keys of small order are
refused, and the equation is checked without the cofactor, as in Go and
libsodium. RSA signing uses the Chinese remainder theorem with
constant-time Montgomery multiplication and fixed 4-bit windows, and
checks each signature with the public key before returning it. It is the
slowest operation here (about 1.2 ms for a 2048-bit key; verifying
takes 0.03 ms), so prefer ECDSA or Ed25519 keys where there is a
choice.
minisign signs the BLAKE2b-512 digest of a file with plain
Ed25519, so its check is `ed25519Verify(pub, crypto.blake2b(file), sig)`;
`std.crypto.minisign` reads and writes its key and signature files.

P-256 uses Renes, Costello and Batina's complete formulas, so no input
(the identity, doubling, a point and its negation) needs a branch, and
fixed 4-bit windows with constant-time table lookups. The field and the
group order are 256-bit Montgomery arithmetic in five 52-bit limbs
(R = 2^260), 25519's field is five 51-bit limbs, and RSA and P-384 use
64-bit limbs. Products are 64 by 64 bits with `core.mulHi` and
`core.mulWide` (one `MUL` on x86-64, `UMULH` on arm64); the 51- and
52-bit limbs leave room to add a column of products up without carries,
and RSA's 64-bit ones carry with `core.addCarry`, which compiles to
`ADC` (`ADCS` on arm64).
The base-point tables of Ed25519 and P-256 are computed on first use.
Rough costs on a recent x86-64: X25519 65 µs, Ed25519 signing 32 µs and
verification 120 µs, P-256 ECDH 290 µs, ECDSA signing 100 µs and
verification 370 µs.

### minisign

`std.crypto.minisign` checks and makes the signatures of the minisign
tool, which rsign and Tauri's updater share, so a program can verify its
own updates against a release key (what the minisign-verify crate does).

```bismuth
import std.crypto.minisign;

let key = minisign.PublicKey.parse(fs.readText("release.pub")?)?;
let sig = minisign.Signature.parse(fs.readText("app.tar.gz.minisig")?)?;
key.verify(data, sig)?;                       // key id, signature, trusted comment
print(sig.trustedComment);                    // "timestamp:...\tfile:app.tar.gz\thashed"
let same = minisign.verify(keyText, data, sigText)?;   // all in one; returns the Signature

let sk = minisign.SecretKey.generate()?;      // a random key id
let made = sk.sign(data, "file:app.tar.gz")?; // prehashed (ED); prehash = false for legacy Ed
fs.write("app.tar.gz.minisig", made.encode())?;
fs.write("release.pub", sk.publicKey().encode())?;
```

A public key is the base64 of `Ed`, an 8-byte key id and the 32-byte
Ed25519 key, after an `untrusted comment:` line. A signature file is four
lines: the untrusted comment; the base64 of the algorithm (`ED` signs the
BLAKE2b-512 digest of the message, as minisign does by default; `Ed`, the
legacy form, the message itself), the key id and the 64-byte signature;
`trusted comment: ` and its text; and the global signature, Ed25519 over
the signature and that text, so the trusted comment cannot be changed
either. `parse` takes the files as minisign writes them (LF or CRLF), the
base64 of the whole file (as Tauri's updater keeps signatures and keys in
its configuration and manifests), a signature without its untrusted
comment line, and, for a public key, the bare key line.

`verify` checks the key id first, then the signature, then the global
signature, and each failure has its own `MinisignError`:
`KeyIdMismatch(signedBy, key)` (made by another key; ids in hex as
minisign prints them), `BadSignature` (the message changed),
`BadTrustedComment`, `LegacyNotAllowed` (with `allowLegacy = false`),
`Malformed(why)`, `UnsupportedAlgorithm(alg)`; `message()` words them,
"minisign: bad signature: the message does not match".
`verifyPrehashed(digest, sig)` and `signDigest(digest, ...)` take the
digest of a file hashed as it streams (`crypto.Blake2b`).
`Signature.signedBy(key)` compares key ids only, to tell a feed signed
by another key apart before downloading anything.

`SecretKey` signs: `generate()`, `fromEd25519(key, keyId)`, or `parse`
of a key file saved without a password (`minisign -G -W`; its `encode()`
writes one, with its checksum). Signing is deterministic, so a key gives
the same bytes minisign does. An empty trusted comment becomes
`timestamp:<Unix seconds>`; comments are one line. Password-protected
secret keys (scrypt, then XOR with the derived key) are refused with
`EncryptedKey`: std has no scrypt.

### Key encodings

| Item | What |
|---|---|
| `PrivateKey` | `Ed25519(Ed25519Key)`, `X25519(X25519Key)`, `P256(P256Key)`, `Rsa(RsaPrivateKey)`; `parsePkcs8(der)` (PKCS#8 and RFC 5958's v2), `parsePem(text)` ("PRIVATE KEY", "EC PRIVATE KEY" or "RSA PRIVATE KEY"), `encodePkcs8()`, `encodePem()`, `publicKey()` |
| `PublicKey` | `Ed25519(bytes)`, `X25519(bytes)`, `P256(P256PublicKey)`, `P384(P384PublicKey)`, `Rsa(RsaPublicKey)`; `parseSpki(der)` (SubjectPublicKeyInfo; RSA as rsaEncryption or RSASSA-PSS), `parsePem(text)` ("PUBLIC KEY" or "RSA PUBLIC KEY"), `encodeSpki()`, `encodePem()` |

Encodings follow RFC 5208, 5915, 5280, 8017 and 8410, and match what
OpenSSL writes byte for byte. Encrypted PEM keys are not supported.

Fallible calls return `CryptoError`: `Invalid(why)` for an argument out
of range (a key or nonce of the wrong size, an HKDF length over 255 hash
lengths, fewer than one PBKDF2 iteration), `Random(why)` if the OS gives
no random bytes, `Auth` when a sealed message does not authenticate, and
`Encoding(why)` for a key, point or DER/PEM structure that does not
decode. Signature checks return `bool`. `==` on two `List<u8>` compares
their storage, not their bytes; compare digests and MACs with
`constantTimeEqual`.

SHA-256 and SHA-512 run their 64 and 80 rounds unrolled, with the state in
registers and the round constants as immediates (`std/crypto/sha2.bi`,
generated by `tools/gensha2.py`); `bench/sha256` compares them with C and
Rust. SHA-256 (and SHA-224, and HMAC, HKDF and PBKDF2 on them) uses the
CPU's SHA instructions where it has them, asked once per process: x86-64's
SHA extensions (`sha256rnds2`, `sha256msg1`, `sha256msg2`; CPUID leaf 7,
with SSSE3 and SSE4.1) and arm64's SHA2 (`sha256h`, `sha256h2`,
`sha256su0`, `sha256su1`; `AT_HWCAP`'s `HWCAP_SHA2` on Linux, every
Apple Silicon CPU), about 5.5 times the unrolled code's speed on long
messages (std/crypto/sha256hw*.bi). `useShaInstructions(false)` keeps it
to the unrolled code (for tests and benchmarks; `true` turns them back on
where the CPU has them; it returns whether they are on). BLAKE2b's rounds (`tools/genblake2b.py`), the 25519 field products
(`tools/gen25519.py`) and the 256-bit Montgomery multiplications
(`tools/genmont.py`) are generated the same way. SHA-1 is there for
protocols that still need it (the WebSocket handshake, TOTP); it is
broken for collision resistance, so don't use it for new signatures or
integrity checks. For games and simulations, `std.rand` is faster and
can be seeded.

`std.uuid` makes RFC 9562 UUIDs: `uuid.v4()` (122 random bits) and
`uuid.v7()` (Unix milliseconds, then a 12-bit counter, then 62 random
bits; each is greater than the last one the process made, even when the
clock steps back, so they sort by creation and keep database indexes
compact). `uuid.parse` reads the 36-character form, 32 bare hex digits,
`{braces}` or `urn:uuid:`, in either case. Both generators panic if the OS
has no random bytes, which no supported OS does.

### DER and PEM

`std.encoding.der` reads and writes ASN.1 DER (X.690), for keys,
certificates and TLS. Reading is strict: definite and minimal
lengths, minimal integers, tag numbers below 31. A `Reader` keeps one
copy of the bytes, shared by the readers of nested elements, so it may
outlive the list it was made from.

```bismuth
import std.encoding.der;

var r = der.Reader.new(bytes);
var spki = ref r.readSequence()?;
var alg = ref spki.readSequence()?;
let oid = ref alg.readOid()?;                  // "1.2.840.10045.2.1"
let key = ref spki.readBitString()?;
spki.finish()?;                                // nothing left over

var w = der.Writer.new();
ref w.begin(der.SEQUENCE);
ref w.writeInt(0);
ref w.writeOid("1.3.101.112");
ref w.end();                                   // fills in the length
let out = ref w.take();
```

| Item | What |
|---|---|
| `Reader` | `new(bytes)`; `read() : Element`, `expect(tag)`, `readSequence`, `readSet`, `readConstructed(tag)`, `readOptional(tag) : Element?`, `readExplicit(n) : Reader?`; `readInt` (fits in an `int`), `readUnsigned` (big-endian magnitude), `readBool`, `readNull`, `readOctetString`, `readBitString` (whole bytes), `readBits` (bytes and unused bits), `readOid` (dotted text), `readString` (UTF8, Printable, IA5, T61 or BMP); `peekTag`, `skip`, `isEmpty`, `remaining`, `offset`, `finish` |
| `Element` | `tag`, `offset`, `len()`, `bytes()` (the contents), `raw()` (the whole encoding, as a signature covers it), `reader()` |
| `Writer` | `new()`; `begin(tag)` / `end()` for SEQUENCE, SET and `explicit(n)`; `writeInt`, `writeUnsigned`, `writeBool`, `writeNull`, `writeOctetString`, `writeBitString`, `writeOid`, `writeUtf8`, `writePrintable`, `writeRaw(tag, body)`, `writeElement(encoded)`; `len`, `bytes`, `take` |
| Tags | `BOOLEAN` `INTEGER` `BIT_STRING` `OCTET_STRING` `NULL` `OID` `ENUMERATED` `UTF8_STRING` `PRINTABLE_STRING` `T61_STRING` `IA5_STRING` `UTC_TIME` `GENERALIZED_TIME` `BMP_STRING` `SEQUENCE` `SET`; `explicit(n)`, `implicit(n)`, `tagName(tag)`; `oidText`, `oidBytes` |

Errors are `DerError.Malformed(offset, why)`, with `message()`.

`std.encoding.pem` armors bytes as RFC 7468 text: `pem.encode(label,
bytes) : str` (64-character lines), `pem.decode(text) : Block !
ParseError` (the first block; text around it is skipped) and
`pem.decodeAll(text) : List<Block>` (a certificate chain); `Block` has
`label` and `bytes`. Blocks with RFC 1421 headers (OpenSSL's encrypted
keys) are refused.

### X.509 certificates

`std.crypto.x509` parses certificates (RFC 5280), verifies chains the way
webpki does, finds the operating system's trust anchors, and builds and
signs certificates and certificate requests (what rcgen does), so tests
and ACME clients need no `openssl` command.

```bismuth
import std.crypto;
import std.crypto.x509;

let chain = x509.parsePemAll(fs.readText("chain.pem")?)?;      // leaf first
let roots = x509.systemRoots()?;                                // or x509.Roots.fromPem(caText)?
let path = x509.verify(chain[0], chain, roots, { name: "example.com" })?;

let caKey = crypto.PrivateKey.P256(crypto.P256Key.generate()?);
let ca = x509.selfSigned({ subject: x509.Name.cn("Test CA"), isCa: true }, caKey)?;
let key = crypto.PrivateKey.Ed25519(crypto.Ed25519Key.generate()?);
let leaf = x509.issue({ subject: x509.Name.cn("localhost"), dnsNames: ["localhost"],
                        ipAddresses: ["127.0.0.1"] }, key.publicKey(), ca, caKey)?;
let request = x509.csr(x509.Name.cn("example.com"), ["example.com"], key)?;   // PKCS#10 DER
```

| Item | What |
|---|---|
| `Certificate` | `raw` (DER), `tbs`, `version`, `serial`, `sigAlg` (OID), `signature`, `issuer` and `subject` (`Name`), `notBefore` and `notAfter` (Unix seconds), `spki`, `dnsNames`, `ipAddresses` (4 or 16 bytes), `emails`, `uris`, `isCa`, `pathLen`, `keyUsage` (`KU_*` bits, -1 when absent), `extKeyUsage` (`EKU_*` OIDs), `subjectKeyId`, `authorityKeyId`, name constraints (`permitted`, `excluded`: `Subtree`s), `unhandledCritical`; `publicKey()` (decoded on demand), `spkiSha256()` (the usual pin), `fingerprint()`, `isSelfIssued()`, `validAt(t)`, `encodePem()` |
| `Name` | `raw`, `attrs` (`Attribute { oid, value }`); `cn(text)`, `of(attrs)`, `get(oid)`, `commonName()`, `str()` ("CN=..., O=..., C=...") |
| `verify` | `verify(leaf, intermediates, roots, VerifyOptions { name, time, purpose, maxIntermediates })` returns the path, leaf first and anchor last; `Purpose` `.ServerAuth` (default), `.ClientAuth`, `.Any` |
| Names | `verifyName(cert, name)`: DNS names against dNSName entries (lowercased, one trailing dot dropped, `*` only as a whole leftmost label matching one label), IPv4 and IPv6 addresses (also `[...]`) against iPAddress entries; the common name is not used |
| `Roots` | `new()`, `fromPem(text)`, `add(cert)`, `addPem(text)`, `addAll(roots)`, `len()`; `systemRoots()` |
| Building | `Template { subject, serial, notBefore, notAfter, dnsNames, ipAddresses, emails, uris, isCa, pathLen, keyUsage, extKeyUsage, permitted, excluded }`; `selfSigned(t, key)`, `issue(t, subjectKey, issuerCert, issuerKey)`: Ed25519, ECDSA P-256 with SHA-256, or RSA PKCS#1 v1.5 with SHA-256 signatures, with basicConstraints, keyUsage, subjectAltName, key identifiers and name constraints; `csr(subject, dnsNames, key)` and `parseCsr(der)` (checks the request's signature) |
| Signatures | `verifySignature(key, algOid, params, message, sig, strict)`: RSA PKCS#1 v1.5 and RSASSA-PSS (SHA-256/384/512), ECDSA with P-256 or P-384, Ed25519 |

Verification builds a path depth first from the leaf through the given
intermediates (in any order, the leaf among them or not) to a trust
anchor, trying each candidate issuer whose subject matches (and whose key
identifier agrees) until one path passes every check: signatures (SHA-1
and RSA keys under 2048 bits are refused), validity at `time` for every
certificate but the anchor, `basicConstraints` CA and `keyCertSign` for
issuers, `pathLenConstraint`, extended key usage on the leaf and on
issuers that carry one, DNS and IP name constraints from issuers and
anchors, and no critical extension that is not understood. A leaf that is
itself one of the roots is trusted as it is (a pinned self-signed
certificate). Errors are `X509Error` (`Expired`, `NotYetValid`,
`UnknownIssuer`, `BadSignature`, `NameMismatch`, `NotCa`, `PathTooLong`,
`UsageMismatch`, `UnhandledCritical`, `NameConstraints`,
`UnsupportedSignature`, `Malformed`, `Io`) with `message()`.

`systemRoots()` loads the trust store once per process: the files and
directories named by `SSL_CERT_FILE` and `SSL_CERT_DIR` when set (on every
OS), else the first CA bundle found among the usual Linux paths
(`/etc/ssl/certs/ca-certificates.crt`, `/etc/pki/tls/certs/ca-bundle.crt`,
...) or the certificate directories; on macOS the system anchors from
Security.framework (loaded at run time) plus certificates an
administrator or the user trusts as roots without conditions; on Windows
the ROOT certificate store (which Windows fills on demand, so a root never
used on the machine can be missing). Public keys are decoded when a
signature is checked, so loading a store of a hundred RSA roots takes
about 10 ms.

## json

`import std.encoding.json;` then `json.parse(text)`. Parsing is one pass
over the bytes: strings without escapes are copied in one go, nothing is
allocated per character, and numbers are read without going through text
again where the result is exact. It is strict RFC 8259: no comments,
trailing commas, leading zeros, `NaN` or single quotes; strings must be
valid UTF-8 with control characters escaped, and `\u` escapes must pair
their surrogates. Nesting stops at 512 levels. A `JsonError` is
`Syntax(line, col, message)` or, from `decode`, `Mismatch(line, col,
message)`; `e.message()` is `"line:col: message"`, the column counted in
characters.

A number without a fraction or an exponent that fits in an `int` is an
`Int`; every other number is a `Float` (a number too large for an f64 is an
error). Floats are written in the shortest form that reads back as the
same f64, as serde_json writes them: positionally from `0.00001` up to 16
integer digits (`0.1`, `1.0`, `1000000000000000.0`), else with an
exponent without padding (`1e-7`, `-2.5e-8`, `1e+16`, `1.5e+16`), so writing
what was parsed gives the same values back; NaN and the infinities, which JSON can't hold, are written as
`null`. Objects keep their members in order; a repeated key keeps both
members, and lookups find the last, as JavaScript does.

```bismuth
let doc = json.parse(text)?;
let mesh = doc["nodes"][0]["mesh"].int() ?? -1;   // a missing key or index gives Null
for node in doc["nodes"] { print(node["name"].str() ?? "unnamed"); }
var o = json.object();
ref o.set("hp", json.Value.Int(100));
print(json.pretty(o));
```

`v["key"]` and `v[i]` return a place, not a copy, so walking a large
document costs nothing; on a missing member they give a shared `Null`
(assign through `set`, not through a missing index). `v[i]` on an object is
its i-th member's value. `items()` and `members()` are slices of an
array's elements and an object's members.

**Typed encoding.** `json.encode(value)` and `json.decode<T>(text)` go
straight between JSON text and Bismuth values, through code the compiler
generates for each type (`core.serialize` and `core.deserialize`, which
call a `Writer`'s and a `Reader`'s methods). A struct is an object of its
non-`priv` fields by name; `List<T>`, tuples, fixed arrays, vectors and
matrices are arrays; `Map<str, V>` is an object; an enum without data is
its variant's name; `T?` is `null` or the `T`; a `char` is a one-character
string. Decoding, a missing field takes its default value (a `T?` field
without one is `null`), `priv` fields take their defaults, members the
struct has no field for are skipped, and a missing field without a
default, a duplicate key, a number outside the field's type or ranged type,
or an array of the wrong length is a `Mismatch` error at its position.
Types declared in std take the same rules (a `webview.Geometry` is an
object of its fields), except that a std struct with a `priv` field is a
handle (`Shared<T>`, `Chan<T>`, `StrBuf`, a `webview.Window`), not data.
Such handles, pointers and maps with non-`str` keys have no JSON form and
are compile errors.

**Attributes.** `@serde(...)` on a struct, field, enum or variant shapes
the JSON as serde's attributes do in Rust:

```bismuth
@serde(renameAll: "snake_case")
struct Container {
    str containerId;                          // "container_id"
    @serde(rename: "Names") List<str> names;
    @serde(alias: "img") str image;           // also read from "img"
    @serde(skipIfNull) str? status = null;    // left out when null
    @serde(skipIfEmpty) List<str> labels;     // left out when empty
    @serde(default) int restarts;             // missing: 0 ("" for a str)
    @serde(skip) int cache = 0;               // never written, read as its default
    @serde(flatten) Meta meta;                // Meta's members inline
    @serde(flatten) Map<str, json.Value> rest;   // every other member
}

@serde(tag: "type", renameAll: "snake_case")
enum Event {
    Started(str id, int pid),                 // {"type": "started", "id": ..., "pid": ...}
    Stopped(StopInfo),                        // StopInfo's members beside "type"
    @serde(other) Unknown,                    // any other "type"
}
```

| On | Options |
|---|---|
| struct | `renameAll: "..."` (`lowercase`, `UPPERCASE`, `PascalCase`, `camelCase`, `snake_case`, `SCREAMING_SNAKE_CASE`, `kebab-case`, `SCREAMING-KEBAB-CASE`), `denyUnknownFields` (an unknown member is an error), `default` (every missing field takes its default or empty value) |
| field | `rename: "..."`, `alias: "..."` (read under another name too; repeatable), `skip`, `skipIfNull` (a `T?`), `skipIfEmpty` (a `str`, `List`, `Map` or `T?`), `default` (missing: the field's default, else `0`, `false`, `""`, empty, `null`, or a struct of those), `flatten` (a struct's members, or a `Map<str, V>` that takes the members no field does) |
| enum | `renameAll: "..."` (variant names), `tag: "type"` (internally tagged), `tag:` with `content: "data"` (adjacently tagged: `{"type": "Name", "data": ...}`), `untagged` (the data alone; reading tries the variants in order) |
| variant | `rename: "..."`, `alias: "..."`, `other` (a variant without data taken for a tag no variant has) |
| a variant's named value | a field's options but `flatten`: `Text(@serde(rename: "body") str text, @serde(skipIfNull) str? author)` |

An enum with data and no attribute is externally tagged: `{"Name":
data}`, and `"Name"` for its variants without data. A variant's data is
the value itself when it is one unnamed value (`Text(str)`), an object of
its names when they are named (`Move(int x, int y)`), and an array when
there are several unnamed values. An internally tagged variant must hold
named values, one struct or one `Map<str, V>`, whose members go beside
the tag; the tag may come anywhere in the object. Misused attributes are
compile errors that name the options the place takes. `bic ts` writes
TypeScript declarations for the types marked `@ts`, in this shape, for a
web client ([tooling.md](tooling.md#bic-ts)).

**Dynamic values in typed data.** A `json.Value` field (or `List<Value>`,
`Map<str, Value>`) takes any JSON. A `json.RawValue` field keeps its
member's text as it was (`raw.text`, `raw.value()` parses it) and is
written back verbatim. Any type can do the same by declaring
`fn serialize<W>(view self, ref W w)` and a static
`fn deserialize<R>(ref R r, ref T out) : bool`, which `encode` and
`decode` then call instead of generated code (with any writer or reader,
so the same methods serve `std.encoding.yaml`). `toValue(x)` and
`fromValue<T>(v)` convert between typed values and `Value`. A `Value`
that doesn't fit `T` gives a `JsonError.Mismatch` worded as Rust's
`serde_json::from_value` words it, so an API ported from Rust answers its
clients with the same text: ``missing field `name` ``, ``duplicate field
`name` ``, ``unknown field `x` ``, ``unknown variant `Green` ``,
``invalid type: string "3", expected i64`` (the value as serde shows it:
`null`, ``boolean `true` ``, ``integer `5` ``, ``floating point `1.5` ``,
a quoted string, `sequence` or `map`; what was expected as `a string`,
`a boolean`, `i64`, `u8` and the other integer types by their range,
`f64`, `u64`, `a character`, `a sequence`, `a map`, or `string or map`
for an enum), ``invalid value: integer `300`, expected u8``, `invalid
length 1, expected a tuple of size 2`. Its line and column are 0, as a
`Value` has no text to point into, and `message()` is the text alone.
Where serde names the struct (`expected struct User`) or lists the fields
or variants it knows (`, expected one of ...`), this says `a map`, and
stops after the name. `decode` keeps its own wording, with the position
in the text.

**Value.** Besides lookups and building, `v.equals(w)` compares as JSON
means it (numbers by value, `1 == 1.0`; objects whatever their order),
`v.get(key) : Value?` copies a member, and `v.pointer("/nodes/0/name")`
follows an RFC 6901 JSON Pointer.

**Streaming.** `json.Pull` reads a document as it arrives, in pieces of
any size, and hands out tokens without building the tree:

```bismuth
var p = json.Pull.new();                 // Pull.new(many: true) for JSON Lines
ref p.feed(chunk);                       // each piece as it comes (feedStr for text)
loop {
    match ref p.next()? {
        NeedMore => break,               // feed more, or finish() at the end of input
        End => break,
        Key(k) => ..., Str(s) => ..., Int(i) => ..., Float(f) => ..., Bool(b) => ...,
        Null => ..., BeginObject => ..., EndObject => ..., BeginArray => ..., EndArray => ...,
    }
}
```

Tokens come only once complete (a number split between pieces waits for
the next), the grammar is checked as they come, and errors carry the
line and column. `depth()` is the nesting at the current token. Writing,
`Writer.drain()` hands over the text written so far and keeps the
writer's place, so a large document goes out in pieces; `writeRaw`
writes text that is already JSON.

## audio

`import std.audio;` plays sounds through the OS's own audio API and mixes
them on a thread of its own.

```bismuth
let dev = audio.open()?;                       // the default output device
defer dev.close();
let hit = audio.load("hit.wav")?;
let v = dev.play(hit, { volume: 0.8, pan: -0.3, pitch: 1.2 });
let music = dev.play(audio.load("theme.wav")?, { looping: true });
music.setVolume(0.5);
```

**Backends.** On Linux, PulseAudio's simple API (`libpulse-simple.so.0`,
which PipeWire's pulse server also serves), then ALSA (`libasound.so.2`,
the `default` device, or `Options.device`, or `BISMUTH_ALSA_DEVICE`);
both are loaded at run time, so a program starts
on any machine and `open` returns `AudioError.NoDevice` where there is no
sound server or card. On Windows, WASAPI in shared mode, event-driven,
through COM vtable calls (`std.interop.com`); the stream asks for 32-bit
float at the mixer's rate and lets Windows convert, else takes the
device's mix format. macOS has no device backend yet (`Unsupported`).
`BISMUTH_AUDIO=pulse|alsa|wasapi|offline` overrides `Auto`. ALSA's error
messages are silenced while it is in use.

**The mixer** runs at the device's rate (48 kHz by default) in 32-bit
float stereo and converts each pass to the device's format: `F32`, `I16`,
`I24`, `I32` or `U8`, little-endian, any channel count (mono is the
average of left and right; channels past the second are silent). Each
voice reads its source with a 32.32 fixed-point position, so pitch
changes and loops are exact, with linear (default) or Catmull-Rom cubic
interpolation (`interp: audio.Interp.Cubic`). Mono sources pan with
constant power (centre is -3 dB per side), stereo ones by balance.
Volume, pan and stop ramp over one pass (`periodFrames`, 10 ms by
default), so they don't click. Output is clipped to -1..1. The mixing
path calls no libm, so a mix is bit-identical on every OS.

**Voices and streams** are handles: a voice that has finished, been
stopped or whose device was closed ignores every call, and the device's
memory goes with its last handle. `queue(channels, rate)` returns a
`Stream` that plays what `push` gives it (silence while empty, ending
after `finish` once drained); `stream(channels, rate, feed)` calls `feed`
on the mixer thread to append up to `frames` frames, until it returns
false. A stream has its own rate and is resampled like a sound. The
mixer's mutex is held while a pass is mixed, so device calls wait at most
that long, and `feed` must not call into the device.

**Offline rendering.** `audio.offline(rate, channels, sample)` (or
`backend: Offline`) opens a device whose backend renders into memory:
`render(frames)` mixes that many frames, `output()` takes the bytes.
Tests use it to check mixes byte for byte without a sound card. Other
outputs extend `Backend` (`open`, `wait`, `lock`, `commit`, `close`,
`name`, `realtime`) and open with `openWith([new MyBackend { ... }])`.

**Decoding.** `decodeWav` reads RIFF WAVE: PCM at 8, 16, 24 and 32 bits,
32- and 64-bit float, and `WAVE_FORMAT_EXTENSIBLE`. `decodeFlac` reads
FLAC (RFC 9639) at 4 to 32 bits and 1 to 8 channels, bit-exact (a 32-bit
sample is rounded to a float's 24 bits); a frame whose CRC fails is
skipped, and a file cut short plays up to the cut. `decodeMp3` reads
MPEG-1, 2 and 2.5 Layer III (a port of minimp3, giving its samples):
joint, intensity and dual-channel stereo, CRCs, free-format bit rates,
the bit reservoir; a Xing or Info frame is skipped and its LAME encoder
delay and padding are cut, so loops are gapless; ID3v1 and APE tags at the
end are skipped; Layer I and II files are `Unsupported`. `decodeVorbis`
reads Ogg Vorbis (a port of stb_vorbis, within rounding of libvorbis):
floor 1, residues 0 to 2, coupling, any block sizes, trimmed at both ends
by the granule positions as libvorbis trims; Ogg pages whose CRC fails
are dropped, and only the first logical stream (and the first link of a
chained file) is read; floor 0, which no encoder has written since 2001,
is `Unsupported`. More than two channels fold to the front left and right
(the first two, but Vorbis's third for its 3, 5 to 8 channel layouts).
`decode` and `load` pick the format by signature (an ID3v2 tag before it
is skipped).

**Decoders.** `decoder(bytes)` and `openDecoder(path)` give a
`Shared<Decoder>` that holds the compressed file and decodes it a block at
a time: `read(ref out, frames)` appends up to `frames` interleaved frames
and returns how many (0 at the end), `rewind` starts over, `readAll`
makes the rest a `Sound`; `channels`, `rate`, `frames` (-1 when the file
does not say) and `kind` (`"wav"`, `"flac"`, `"mp3"`, `"vorbis"`). `Device.playStream(d,
opts)` plays one as a `Stream` that the mixer thread fills a pass at a
time (`looping` rewinds it; `Voice.setLoop` changes that), so music is
never decoded whole; a decoder error ends the stream.

## csv

`import std.encoding.csv;` reads and writes comma-separated values as RFC
4180 defines them: quoted fields, doubled quotes inside them, line breaks
inside quotes, and rows ending in CRLF or LF. `Options` sets the
`delimiter` and `quote` (ASCII `char`s), `header: true` (the first record
names the columns) and the `mode`; blank lines are skipped and a UTF-8
byte order mark is dropped.

```bismuth
var r = csv.parse(text, { header: true });
let names = ref r.header()?;                // Slice<str>
let age = r.column("age") ?? 1;
for row in r {
    total += row[age].parseInt().or(0);      // row[i] is a Field
    print(row.field("name")?.str() ?? "?");
}
if r.err() != null { /* a for loop stops at the first error */ }
```

A `Row` and its `Field`s are views into the reader's buffer: reading a
record allocates nothing per field, and a field is copied only when
`str()` (or `toList()`) asks for it. A field with doubled quotes is
unescaped into a scratch buffer the reader reuses. A row is valid until
the next read. `ref r.read()` returns `Row? ! CsvError`, null at the end;
`for row in r` stops at the first error and keeps it in `r.err()`. A
`Reader` is a handle, so the copy a `for` loop takes shares its position.
`fromFile(f)` reads an open `fs.File` in chunks, holding only the record
being read; a record that runs past a chunk is read again once the next
chunk is in.

`.strict` (the default) rejects a quote inside an unquoted field, text
after a closing quote, a quote that is never closed, and a record whose
field count differs from the first record's. `.lenient` takes stray
quotes as text, runs an unclosed quote to the end of the input and allows
ragged rows. Errors are `CsvError.Syntax(line, col, why)`, or an
`IoError` from the file; `message()` gives `line:col: why`, with the
column counted in characters.

`writer(opts)` builds text in memory. `field(s)` quotes a field when it
holds the delimiter, the quote or a line break; `value(v)` writes any
value as `"{v}"` would; `endRow()` ends the row with `\n` (`\r\n` with
`crlf: true`). A row of one empty field is written `""`, so it doesn't
read back as a blank line. `take()` hands over the text and `flush(ref f)`
writes it to a file.

## toml

`import std.encoding.toml;` reads TOML 1.0 into a tree and writes it back.

```bismuth
let cfg = toml.parse(text)?;                 // Table ! TomlError
let port = cfg.at("server.port")?.asInt() ?? 8080;
for k, v in cfg { print("{k} = {v.str()} ({v.typeName()})"); }
var server = cfg.getTable("server") ?? toml.Table.new();
ref server.set("port", toml.Value.Integer(9090));   // cfg shares server
print(toml.encode(cfg));
```

`Value` is an enum of `String`, `Integer` (`int`), `Float` (`f64`, with
`inf` and `nan`), `Boolean`, `DateTime`, `Array` and `Table`. A
`DateTime` keeps its `kind` (`OffsetDateTime`, `LocalDateTime`,
`LocalDate`, `LocalTime`) beside a `std.time` `Date` and `Time` and an
offset in seconds; `local()` gives a `time.DateTime` and `instant()` the
`time.Timestamp` of an offset date-time. A `Table` keeps keys in the
order they were added. Tables and arrays are handles: copying one shares
it, so walking a tree copies nothing and changes made through a copy show
in the document; `deepCopy()` copies. `equals` compares deeply.

The parser checks all of TOML 1.0: the four string kinds (a newline right
after the opening `"""` or `'''` is dropped, and CRLF in a multi-line
string becomes LF), integers with `_` and `0x`, `0o` or `0b` (range
checked), floats, dates and times (ranges, leap years), UTF-8, and the
rules on defining tables: no duplicate keys, no `[table]` twice, no
`[table]` for a table made by dotted keys, no dotted keys into a table
made by a header, nothing added to inline tables or to arrays not made by
`[[ ]]`. Errors are `TomlError.Syntax(line, col, why)`. It passes every
TOML 1.0 case of [toml-test](https://github.com/toml-lang/toml-test)
v1.6.0 (185 valid, 371 invalid); they are in `tests/run/toml-test`,
under its MIT licence (`tests/run/toml-test/LICENSE`).

`encode` writes a table's plain values first, then a `[header]` for each
table inside it and `[[header]]`s for each array of tables; a table
holding only tables gets no header of its own. Comments and the input's
layout are not kept. What it writes parses back to an equal tree.

## yaml

`import std.encoding.yaml;` reads and writes YAML 1.2: docker compose
files, CI workflows, Kubernetes manifests.

```bismuth
let doc = yaml.parse(text)?;                          // Value ! YamlError
let image = doc["services"]["web"]["image"].str() ?? "";
let cfg = yaml.decode<Compose>(text)?;                // into structs
print(yaml.encode(cfg));                              // block style

var p = yaml.Parser.new(text);                        // events, one at a time
loop {
    let e = ref p.next()?;
    if e.kind == yaml.EventKind.StreamEnd { break; }
}
```

The whole language is read: block and flow collections, the five scalar
styles (plain, single and double quoted with every escape, literal `|` and
folded `>` with chomping and indentation indicators), multi-document
streams, comments, `%YAML` and `%TAG` directives, anchors and aliases,
tags (`!!str`, `!local`, `!e!handle`, `!<verbatim>`) and complex keys.
Errors are `YamlError.Syntax(line, col, message)` (and `Mismatch` from
`decode`), with 1-based positions. It passes every case of the
[yaml-test-suite](https://github.com/yaml/yaml-test-suite) (data of
2022-01-17, which includes every example of the YAML 1.2 spec): the 308
valid streams give the suite's events exactly, the 94 invalid ones are
errors.

**Values.** Plain scalars are resolved with the YAML 1.2 core schema:
`null`, `~` and empty are `Null`; `true`/`false` (three spellings)
`Bool`; decimal, `0o` and `0x` integers `Int`; floats with `.inf` and
`.nan` `Float`; the rest, and every quoted or block scalar, `Str`. Explicit
`!!str`, `!!int`, `!!float`, `!!bool` and `!!null` tags are obeyed; a node
with a tag outside the core schema is `Tagged(tag, [value])`. Merge keys
(`<<: *defaults`, `<<: [*a, *b]`) are applied, the mapping's own keys
winning. Mappings keep their order, and keys may be any value (`v["k"]`
finds string keys). Aliases are shared while parsing (`load` gives each
document as a graph of nodes); building a tree expands them, and
`Options.maxAliasNodes` (a million by default) caps how many nodes that
may add, which stops billion-laughs documents with an error.

**Typed.** `encode` and `decode<T>` follow the same rules and `@serde`
attributes as `std.encoding.json` (see [json](#json)), through the same
compiler-generated code. Reading, a field takes what YAML allows it: a
`str` field takes any scalar's text (`version: 3.8` reads as `"3.8"`), a
`bool` field also takes `yes`/`no`/`on`/`off`, and an empty value is an
empty `str`, `List` or `Map`. A `yaml.Value` field takes any node, so
loosely typed parts of a file (a compose `command` that is a string or a
list, a workflow's `on:`) stay as they are.

**Writing.** `stringify` and `encode` write block style with two-space
indentation: strings plain when they read back as the same string (and
not as `yes`, `no`, `on` or `off` to YAML 1.1 readers either), single
quoted when printable, multi-line ones as literal blocks, the rest
double quoted with escapes; empty collections as `[]` and `{}`; shared
nodes with anchors. The `Emitter` also takes events, and documents read
with `load`: their scalar styles, flow collections, anchors, tags and
directives are kept, so what it writes parses back to the same events
(checked against every valid case of the test suite). Comments are not
kept.

## regex

`import std.text.regex;` then `let re = regex.compile(r"(\d+)-(\d+)")?;`.
Write patterns as raw strings: in a normal string `{` interpolates. The
guide is [wiki/Regular-Expressions.md](../wiki/Regular-Expressions.md).

| Item | What |
|---|---|
| `compile(str) : Regex ! RegexError` | parses and compiles a pattern; `RegexError.Syntax(pos, message)` with `pos` a byte offset in the pattern, or `TooBig(size)` past 50,000 instructions; `e.message()` is one line |
| `escape(str) : str` | backslashes the metacharacters, so the result matches the text itself |
| `Regex` | `isMatch(hay)`; `find(hay)` and `findAt(hay, from) : Match?` (the leftmost match at or after byte `from`); `findAll(hay)` (an iterator of `Match`); `captures(hay)`, `capturesAt(hay, from) : Captures?` and `capturesAll(hay)`; `replace(hay, template)`, `replaceAll`, `replaceN(hay, template, n)` (`n < 0`: all), `replaceAllWith(hay, fn(view Captures) : str)`; `split(hay)`, `splitN(hay, n)`; `pattern()`, `groupCount()` (without group 0), `groupNames()` (`""` for unnamed groups), `groupIndex(name) : int?`; `fresh()`. Interpolates as its pattern |
| `Match` | `start`, `end` (byte offsets, `end` exclusive), `text()`, `len()`, `isEmpty()`; interpolates as its text |
| `Captures` | `len()` (groups, group 0 included), `get(i) : Match?`, `name(n) : Match?`, `text(i)` (`""` when the group took no part), `start(i)`, `end(i)` (-1 then), `expand(template, ref StrBuf)` |

Syntax: literals and `\` escapes (`\t \n \r \f \v \a \e`, `\x41`,
`\x{1F600}`, `\u00e9`, any punctuation); `.` (any character but `\n`,
any with `(?s)`); sets `[a-z]`, `[^...]`, with `\d \w \s \D \W \S` and
POSIX classes `[[:alpha:]]`, `[[:^digit:]]` (`alnum alpha ascii blank cntrl
digit graph lower print punct space upper word xdigit`) inside; `^ $`
(text, or line with `(?m)`), `\A \z`, `\b \B`; groups `(x)`, `(?<name>x)`,
`(?P<name>x)`, `(?:x)`; `x|y`; `* + ? {n} {n,} {n,m}` (counts up to 1000),
each lazy with a trailing `?`; flags `(?imsx)`, `(?i:x)`, `(?-i)`. `\d \w
\s \b` and the POSIX classes are ASCII; `(?i)` folds case with Unicode's
simple folds (Latin, Greek, Cyrillic, Armenian, Georgian, Glagolitic,
Coptic, fullwidth and Deseret letters, and the Kelvin and Ångström signs).
Not supported, each refused by `compile` with its own message:
backreferences, look-around, atomic groups, possessive repetition, `\p`
Unicode classes.

Matching is leftmost-first, as in Perl, Python, Go and Rust. `findAll`,
`replaceAll` and `split` skip an empty match that starts where the
previous match ended. Offsets are bytes and always fall between
characters; bytes that are not valid UTF-8 match no set, not even `.`.

Every search is linear in the text. The pattern compiles to a Thompson
NFA (and a reversed copy); a search first looks for a literal prefix, or
a literal every match must contain, with `memchr`; then a lazy DFA, whose
states are built as needed and kept in the `Regex`, runs forward to the
end of the leftmost match and backward over the reversed program to its
start; groups come from a bounded backtracker for short matches and a
Pike VM otherwise. A DFA that would grow past 4096 states drops its cache
and leaves that search to the Pike VM. Copies of a `Regex` share its
caches, so a `Regex` is not for use by two threads at once: give each its
own `fresh()` copy.

## Unicode normalization

`std.text.unicode` puts text in one of Unicode's four normal forms
(UAX #15), as the unicode-normalization crate does, so that strings that
look the same compare the same: `"e\u{301}"` (e and a combining acute)
and `"\u{e9}"` (é) are both `"\u{e9}"` in NFC and both `"e\u{301}"` in
NFD.

```bismuth
import std.text.unicode;

let key = unicode.nfc(name);                         // composed: how most text is stored
let folded = unicode.nfkc("\u{fb01}\u{2075}");        // "fi5": compatibility forms folded
// a sort key without accents: decompose, then drop the marks
var b = StrBuf.new();
for c in unicode.nfd(name).chars() {
    if !unicode.isCombiningMark(c) { ref b.writeChar(c); }
}
if !unicode.isNormalized(input, unicode.Form.Nfc) { ... }
```

| Item | What |
|---|---|
| `Form` | `Nfc` (canonical decomposition, then composition), `Nfd` (canonical decomposition), `Nfkc` and `Nfkd` (the same with the compatibility mappings: ligatures, widths, super- and subscripts, circled and styled letters) |
| `normalize(s, f) : str` | `s` in form `f`; `nfc(s)`, `nfd(s)`, `nfkc(s)`, `nfkd(s)` |
| `isNormalized(s, f) : bool` | the quick check, then a full normalization only when it says `Maybe` |
| `quickCheck(s, f) : QuickCheck` | UAX #15's quick check: `Yes`, `No`, or `Maybe` (marks that might compose with what precedes them) |
| `combiningClass(c) : int` | the canonical combining class, 0 to 254 (0 for starters) |
| `isCombiningMark(c) : bool` | general category Mn, Mc or Me |
| `decompose(c, compat = false) : str` | the full canonical (or compatibility) decomposition of one char |
| `UNICODE_VERSION` | the version of the tables, "15.1.0" |

Text already in the form comes back without a copy: an ASCII string is
returned at once, and otherwise the quick check finds the first
character that may change, so only the text from the starter before it
is decomposed (into code points in canonical order) and, for NFC and
NFKC, recomposed. Hangul syllables are composed and decomposed
arithmetically. Bytes that are not UTF-8 are kept as they are and
combine with nothing.

The tables in `std/text/unicode/normtables.bi` are generated by
`tools/gennormalize.py` from Python's Unicode Character Database, as
`tools/genunicode.py` makes `char.isLetter`'s, and checked by it against
`unicodedata` before they are written: the combining classes as runs, the
combining marks as ranges, and each decomposition mapping one level deep
with a tag for canonical pairs that recompose, other canonical mappings
and compatibility ones. They add about 60 KB to a program that imports
the module, and nothing to one that does not. The first call builds a
two-level lookup table from them (one entry per 128 code points below
U+30000, then blocks of record numbers where any of those 128 has a
class, a mapping or a composition),
with the full decompositions, the composition pairs and the quick-check
properties, which all follow from the mappings. `tests/run/unicode_normalize`
runs the whole of Unicode's NormalizationTest.txt for that version.

## Allocators

Every allocation a program makes goes to one heap: strings, lists, maps,
`new`, closures, tasks, `mem.alloc`, `allocZeroed`, `realloc` and `free`,
and std's `malloc`, `calloc`, `realloc` and `free`. By default that is the
C library's `malloc`. A program can pick another:

- `bic build --allocator=bismuth` (and `bic run`) sends them all to
  `BismuthAllocator`, std's heap for threaded programs such as servers.
  `--allocator=c` is the default spelled out.
- A struct marked `@allocator` receives them all through its static
  functions `alloc(int size) : *u8`, `allocZeroed(int size) : *u8`,
  `realloc(*u8 p, int size) : *u8` and `free(*u8 p)`, which have the C
  library's meanings. A program has one at most, and not together with
  `--allocator`. It must not allocate through the program's heap itself;
  one built on the C library calls `cMalloc` and friends.

```bismuth
@allocator
struct Counting {
    fn alloc(int size) : *u8 unsafe { core.atomicAdd(allocs(), 1); cMalloc(size) }
    fn allocZeroed(int size) : *u8 unsafe { core.atomicAdd(allocs(), 1); cCalloc(1, size) }
    fn realloc(*u8 p, int size) : *u8 unsafe { cRealloc(p, size) }
    fn free(*u8 p) unsafe { cFree(p); }
}

fn allocs() : *int unsafe { core.static<int>("app.allocs") }
```

`cMalloc`, `cCalloc`, `cRealloc` and `cFree` are always the C library's.
Memory that a C library allocates and leaves to the caller to free, or
frees itself, must go through them (std's X11 code frees xcb's replies
with `cFree`). A program that picks no allocator compiles exactly as
before; `--interp` always runs with the C library's heap.

**BismuthAllocator** is mimalloc's design written in Bismuth, with no
lock and no atomic operation on the common path. Memory comes from the
system (`mmap`, `VirtualAlloc`) in 4 MiB segments aligned to their size,
each 64 pages of 64 KiB. Every thread has a heap that owns its segments
and serves each of 44 size classes (16-byte steps to 128 bytes, then four
per doubling up to 64 KiB) from a page run whose free list is linked 4 KiB
at a time. The owning thread frees onto a page's local list; another
thread pushes onto the page's thread-free list with one CAS, and the owner
takes those back when the page runs dry. A page whose blocks have all come
back returns to its segment; a heap keeps one empty segment and unmaps the
rest. Blocks over 64 KiB are runs of pages of their own up to 1 MiB and
mappings of their own above. A freed mapping goes to a cache that all
threads share, of up to 8 blocks and 64 MiB, still mapped and touched,
and a later block over 1 MiB that it fits (at most twice over) reuses it
without a system call or page faults, as mimalloc keeps them: allocating,
touching and freeing 16 MiB takes about 0.1 ms a round instead of 4 ms
(x86-64 Linux).
Blocks are 16-byte aligned. A heap is found
through a pthread key (thread-local storage on Windows), so the tasks
that run on a scheduler worker share its heap, however many are parked;
when its thread ends (the key's destructor, a TLS callback on Windows),
it gives back what it can and the next new thread takes it over with
everything left in it. `BismuthAllocator.heapsInUse()` counts the heaps
threads hold, and `heapsMade()` those and the ones waiting for a thread.

Blocks other threads free into a page wait on the page's thread-free list
until its heap takes them back: when it allocates from that size class
again, or when it is trimmed. A trim takes back every page's thread-free
blocks and gives back the pages whose blocks have all come back,
including the current page of a size class the heap has stopped
allocating from. The task scheduler's workers trim their heaps before
they wait for work and every 100 ms while they are busy, its monitor
trims the heaps of waiting threads (a `parallel for` worker between jobs,
a thread that ended) every second, and a thread that needs a new segment
first trims every waiting thread's heap. So a server whose tasks allocate
on one worker and free on another keeps its resident memory near what it
holds (`bench/alloc_crossthread.bi` shows it). A page given back stays
committed for 100 to 200 ms, so a page freed and soon taken again costs
no system call or page faults; then it goes back to the system
(`madvise(MADV_DONTNEED)`, Windows' `MEM_DECOMMIT`, committed again when
next used). That reserve is for threads at work: a heap keeps at most
4 MiB of such pages, past which a page goes back at once, and a thread
that has waited 100 ms (a worker with nothing to run, in the poller or
not, a thread waiting on a task, a `parallel for` worker between jobs)
gives them all back, as does a thread that ends.
`BismuthAllocator.dirtyBytes()` says how much the heaps keep so. A cached
huge block that no allocation takes for a second is unmapped.

A page that a few live blocks keep in use gives back the rest of itself:
when a trim finds a page (or a medium run) with at most a quarter of its
blocks out, and found it so at the trim before (or its thread has waited
100 ms), the system pages (4 KiB; 16 KiB on Apple silicon) that lie
wholly in its free blocks go back to the system. The free blocks there
leave the page's free list, so no link is kept in memory that was given
back, and come back, committed again on Windows, once the rest of the
page's free blocks have been allocated. Medium runs (blocks of 10 KiB and
up) gain the most; a page of small blocks gains where its live blocks sit
together.

**Where a program's memory is.** These read a counter or two each,
kept where memory is mapped, given back and unmapped, so a server can log
them as often as it likes; they are a moment's view, not a consistent
one, while other threads allocate.

| Figure | What it counts |
|---|---|
| `BismuthAllocator.committedBytes()` | `segmentBytes() + hugeBytes()`: all the memory `BismuthAllocator` holds from the system and has not given back |
| `BismuthAllocator.segmentBytes()` | the heaps' 64 KiB headers and 4 MiB segments, less what was given back (whole 64 KiB pages decommitted or discarded, and the system pages of shrunk pages) and not taken again. Pages mapped but never touched count too, so it bounds from above what the segments keep resident: live blocks, free blocks in pages in use, dirty pages, a heap's spare segment |
| `BismuthAllocator.hugeBytes()` | blocks over 1 MiB, each a mapping of its own, live or in the cache (headers included) |
| `BismuthAllocator.hugeCachedBytes()` | the part of `hugeBytes` in the cache of freed huge blocks (up to 64 MiB, unmapped after a second unused) |
| `BismuthAllocator.dirtyBytes()` | the part of `segmentBytes` in free pages kept committed for reuse |
| `BismuthAllocator.heapsInUse()`, `heapsMade()` | heaps threads hold, and those plus the pool's |

`std.os.schedStats()` gives the task scheduler's side as a `SchedStats`,
all zero before the first `spawn` or `await`:

| Field | What it counts |
|---|---|
| `workers` | worker threads running tasks, blocked ones included, retired ones not |
| `fibers` | fibers made: each is a task's stack and registers, kept for the next task when its own ends, never freed |
| `fibersInUse` | fibers running or parked tasks (`fibers - fibersPooled`) |
| `fibersPooled` | fibers waiting for a task: the shared pool's, with their stacks or (past 1024) without, and those each worker keeps |
| `stacksWarm` | fibers whose stack has been started and not given back whole since (a trimmed stack still counts: it keeps its top pages) |
| `stackBytesReserved` | address space reserved for stacks: on Linux and macOS the slabs of 64 stacks of 1 MiB they are cut from (kept), on Windows 1 MiB per OS fiber |
| `stackBytesCommitted` | what the stacks hold in memory, as measured when each was last trimmed: their resident pages (`mincore` on Linux and macOS, `QueryWorkingSetEx` on Windows); 16 KiB (64 KiB on Windows) for a stack started since and not measured yet. On Windows a stack's commit charge (private bytes) is not lowered by a trim: a stack stays committed down to the deepest it has gone |
| `stackBytesTrimmed` | stack pages given back so far, in bytes (below) |
| `stackHighest` | the deepest any stack has been seen to go, in bytes |
| `stackDepths` | how many fibers' stacks have been seen to go how deep, in 7 buckets: up to 16, 32, 64, 128, 256 and 512 KiB, and 1 MiB |

A fiber's stack holds every page its tasks touch, so one that once ran
deep (a TLS handshake, a big frame) would hold those pages for good. The
scheduler's monitor looks every half second for fibers that have stayed
pooled, or parked on the same wait, through two of its looks, and gives
back their stack pages below where each stopped, less 16 KiB
(`madvise(MADV_DONTNEED)` on Linux and macOS; on Windows
`DiscardVirtualMemory`, which leaves the guard page and the commit as
they are, so the working set drops there but private bytes do not).
`stackBytesTrimmed` counts the pages that were resident when given back. Nothing below that point is in use while the fiber is
stopped, and a fiber is never switched to while its stack is being
trimmed. So a parked connection keeps only the frames it waits in, a
pool that is in use is not touched, and an idle one is trimmed about a
second after it went idle. The trim also measures the stack, which is
where `stackHighest`, `stackDepths` and `deepestStacks` come from: a
task that never waits that long is not measured.

`std.os.deepestStacks(n)` lists the n fibers whose stacks went deepest,
deepest first, as `StackDepth { bytes, spawnSite, inUse }`: `spawnSite`
is the code address of the spawn that started the task running then (its
thunk, named after the spawned function in the executable's symbols;
less the load address, from `/proc/self/maps`, for a position-independent
one), to find which tasks need deep stacks. It walks every fiber, so it
is for diagnostics.

A process's resident memory beyond `committedBytes()` plus the stacks is
then the program's code and data, the threads' own stacks, the C
library's heap (`cMalloc`, and C libraries'), and other mappings.

`std.os.memStats()` gives the system's and the C library's side as a
`MemStats` (-1 where they don't say): `resident` and `peakResident`
(Linux's `VmRSS` and `VmHWM`, macOS's resident size and `maxrss`,
Windows's working set and its peak), `cHeapInUse` and `cHeapMapped`
(glibc's `mallinfo2`, looked up at run time so a program still starts on
a glibc before 2.33; macOS's malloc zones; Windows's process heap, which
UCRT's `malloc` uses). The C heap's figures walk its arenas, so log them,
don't poll them in a loop.

Measured on a shared 4-core Linux x86-64 machine, medians of 7 runs:
`bench/alloc.bi` (eight threads; strings and lists; every sixteenth
round's batch freed by the next thread), and a test of the heaps alone
(eight threads each allocating a million blocks of 8 B to 32 KiB, keeping
a window of 1024 and handing a quarter to the next thread to free):

| heap | bench/alloc | heaps alone | peak memory (heaps alone) |
|---|---|---|---|
| glibc 2.39 `malloc` | 2320 ms | 367 ms | 148 MB |
| `BismuthAllocator` | – | 137 ms | 99 MB |
| mimalloc 2.1 (`LD_PRELOAD`) | 823 ms | 140 ms | 161 MB |
| jemalloc 5.3 | 706 ms | 157 ms | 131 MB |
| tcmalloc 2.15 | 681 ms | 115 ms | 56 MB |

With bic itself picking the heap, the bench workflow measured these on
another 4-core x86-64 machine (best of 5 runs):

| benchmark | default (glibc) | `--allocator=bismuth` | C twin, gcc -O2 | Rust twin |
|---|---|---|---|---|
| alloc | 966 ms | 270 ms | 804 ms | 1947 ms |
| trees | 71 ms | 63 ms | 77 ms | 70 ms |
| strs | 44 ms | 49 ms | 51 ms | 65 ms |
| maps | 220 ms | 209 ms | 206 ms | 259 ms |

On one thread (5 million allocate-and-free pairs, and 60 rounds of
65536 small blocks allocated, then freed) the two heaps take the same
time, about 120 ms. A segment whose pages have all come back is unmapped
unless it is its heap's spare, as is a huge block the cache has no room
for: a
thread that allocates 1.5 million small blocks (143 MB resident) and
frees them leaves the process at 22 MB.

## vt

`import std.term.vt;` emulates a terminal without showing it: feed it what
a program writes to its terminal (a pty's output) and it keeps the screen
as xterm would, for a server that shares terminals, records them or answers
"what is on the screen". It covers what the Rust `vt100` crate does, with
DSR replies, DEC line drawing and a scrollback ring added.

```bismuth
var term = vt.Parser.new(24, 80, 1000);      // rows, columns, rows of scrollback
ref term.process(bytes);                     // List<u8>, any chunking; or processStr
print(term.screen.contents());               // the visible text
let cell = term.screen.cell(0, 0);           // Cell?: text, style, wide
let reply = ref term.takeReplies();          // answers to DSR, DA and size queries, for the program

let shown = term.screen;                     // a copy: rows are shared until written
ref term.process(more);
send(term.screen.stateDiff(shown));          // what turns `shown` into the screen now
```

| Item | What |
|---|---|
| `Parser` | `new(rows, cols, scrollback = 0)`; `process(view List<u8>)`, `processStr(str)`, `processRaw(*u8, n)` (unsafe); `resize(rows, cols)`; `takeReplies() : str` (the bytes to write back to the program, then forgotten), `hasReplies()`; the public field `screen` |
| `Screen`: state | `size()` `rows()` `cols()`; `cursorPosition() : (row, col)` (0-based), `wrapPending()`; `hideCursor` `alternateScreen` `applicationCursor` `applicationKeypad` `bracketedPaste` `focusReporting` `insertMode` `originMode` `autowrap` `newlineMode` `reverseVideo` `cursorStyle`; `mouseMode() : MouseMode` (`None` `Press` `PressRelease` `ButtonMotion` `AnyMotion`), `mouseEncoding() : MouseEncoding` (`Default` `Utf8` `Sgr` `Urxvt`); `scrollRegion() : (top, bottom)`; `title()` `iconName()`; `audibleBells()` `visualBells()` (counts); `pen() : Style` (what new text is written with); `tabStops()` |
| `Screen`: text | `contents()` (rows without trailing spaces, joined by newlines except where a row wrapped into the next, trailing empty rows dropped), `rowText(row)`, `rowWrapped(row)`, `contentsBetween(r0, c0, r1, c1)` (c1 exclusive), `cell(row, col) : Cell?` |
| `Screen`: scrollback | `scrollbackLen()` (row 0 the oldest), `scrollbackLimit()`, `setScrollbackLimit(n)`, `clearScrollback()`, `scrollbackText(i)`, `scrollbackCell(i, col)`, `scrollbackWrapped(i)`, `contentsWithScrollback()` |
| `Screen`: output | `contentsFormatted()` (cursor visibility, cells, cursor and pen, for a fresh terminal: vt100's `contents_formatted`), `contentsDiff(prev)` (the cells that changed, the cursor and pen: `contents_diff`), `stateFormatted()` and `stateDiff(prev)` (everything: also the alternate screen with the primary one under it, scroll region, tab stops, saved cursor, modes, character sets, input modes and title), `inputModeFormatted()` `inputModeDiff(prev)`, `titleFormatted()` `titleDiff(prev)`, `bellsDiff(prev)`, `attributesFormatted()`; each returns a `str` of escape sequences |
| `Screen`: changes | `resize(rows, cols)`, and every operation the sequences reach (`print`, `eraseDisplay`, `scrollUp`, `setPrivateMode`, ...) |
| `Cell` | `text` (the character and its combining marks, `""` when empty), `style`, `wide`, `wideContinuation` (the right half of a wide character), `hasContents()` |
| `Style` | `fg`, `bg` (`Color.Default`, `Color.Idx(u8)` for the 256 palette, `Color.Rgb(r, g, b)`), `bold` `dim` `italic` `underline` `blink` `inverse` `invisible` `strikethrough` |
| `Machine` | the parser alone: `new()`, `step(u8) : Action` (`None` `Print` `Execute` `Esc` `Csi` `Osc` `DcsHook` `DcsPut` `DcsUnhook`), `again()` (step the same byte once more), `ch()` `final()` `prefix()` `paramCount()` `param(i, default)` `rawParam(i)` `isSub(i)` `intermediateCount()` `intermediate(i)` `oscData()` `state()` `inGround()` `reset()` |
| widths | `charWidth(char) : int` (0, 1 or 2), `strWidth(str)`; `UNICODE_VERSION` |

The parser is Paul Williams' DEC ANSI state machine (ground, escape, CSI
entry/param/intermediate/ignore, OSC string, DCS entry/param/intermediate/
passthrough/ignore, SOS/PM/APC) with xterm's additions: BEL ends an OSC
string, `:` subparameters are kept apart from `;` parameters (`38:2::r:g:b`),
and text is UTF-8, decoded as the WHATWG decoder does (each bad byte or
cut-short sequence is one U+FFFD; a sequence cut by ESC or a control
still lets that byte act). CAN and SUB cancel a sequence. Up to 32
parameters of up to 65535 and 8 KiB of an OSC string are kept. Raw 8-bit
C1 bytes are UTF-8 continuation bytes, as in xterm's UTF-8 mode; decoded
U+0080-U+009F are not shown. State carries over between calls, so input
may be split anywhere.

The screen follows xterm (and xterm.js where xterm leaves room):

- **Text.** Autowrap with a pending wrap: a character in the last column
  leaves the cursor there, and the next one wraps (marking the row as
  wrapped). Wide characters (East Asian Wide and Fullwidth, most emoji)
  take two cells; one that would start in the last column wraps whole,
  or without autowrap is dropped. Combining marks, ZWJ and variation
  selectors (width 0) join the character before the cursor. Widths are
  wcwidth's rules over Unicode 15.1 (`tools/genwidth.py` regenerates
  `width.bi`). Overwriting half of a wide character clears the other half.
  `ESC ( 0` / `ESC ) 0` with SI/SO select DEC line drawing.
- **Sequences.** CUU CUD CUF CUB CNL CPL CHA HPA HPR CUP HVP VPA VPR CHT CBT,
  ED (0-3, 3 clears the scrollback) EL ECH ICH DCH IL DL SU SD REP, SGR
  (bold, dim, italic, underline including `4:x`, blink, inverse, invisible,
  strikethrough, 16/256/truecolor in `;` and `:` forms), DECSTBM, SM/RM (4
  insert, 20 newline), DECSET/DECRST 1 5 6 7 9 25 47 66 1000 1002 1003 1004
  1005 1006 1015 1047 1048 1049 2004, DECSC/DECRC (per buffer, as xterm)
  and `CSI s`/`CSI u`, DECSTR, RIS, DECALN, IND NEL RI HTS TBC, DECKPAM/
  DECKPNM, DECSCUSR, OSC 0/1/2 titles, BEL and `ESC g` counted. Replies
  are queued for `takeReplies`: DSR 5 and 6 (`?6` too), DA1, DA2 and
  `CSI 18 t`. Other sequences are parsed and ignored.
- **Edges.** At a pending wrap EL 0, ECH, ICH and DCH change nothing (as
  in xterm.js, tmux and vt100). IL and DL do nothing outside the scroll
  region; inside it they move the cursor to column 0. Rows scrolled off the top of
  a whole-screen region of the primary buffer go to the scrollback; the
  alternate screen keeps none.
- **Resize** keeps the contents without reflowing them: rows are cut or
  padded on the right and rows dropped or added at the bottom, except
  that when the cursor's row would be cut off, rows leave from the top
  (into the scrollback) so the cursor keeps its text. A whole-screen scroll
  region stays whole; another is clamped.

The `*Formatted` outputs are for a fresh terminal of the same size. A wrapped row is drawn through
the margin so the terminal wraps it too and knows the rows are one line,
and a pending wrap is recreated by writing the last column again; both
may turn an empty cell into a space. Background colors are erased with
ECH and EL (background color erase, as xterm and most terminals do).
The `*Diff` outputs expect a terminal in `prev`'s state and change only
what differs: unchanged rows are skipped by identity (a copy shares its
rows) before any cell is compared, and an unchanged screen gives `""`.
`stateDiff` redraws everything after a resize or a switch of buffers.

Performance: rows are contiguous arrays of 16-byte cells in one block
each, printable ASCII is copied a run at a time, scrolling moves row
handles, and rows leaving a full scrollback are reused for the new ones,
so feeding output allocates nothing per byte. 10 MB of text with a
256-color SGR every few words takes about 100 ms, plain text about 40 ms
(x86-64). Copying a `Screen` counts its rows and shares its scrollback,
whose chunks of 64 rows are copied only when written.

## Compiler intrinsics

`mem.*` (unsafe memory: `alloc` `allocZeroed` `realloc` `free` `read` `write` `take` `drop` `copy` `move` `zero` `offset` `cast` `addr` `null` `isNull` `size` `align` `needsDrop` `forget`) and `core.*` (`fmt` `hash` `encode` `decode` `typeName` `fields` `checkIndex` atomics, `static`, `fnAddr`) are what the library is built from. `core.fields<T>()` describes a struct for code that fills one from outside data: a line `<size> <flags>` (`-`, or `v` with a vtable, `p` when packed, `l<off>` for a struct with a drop or clone hook, whose hidden live byte at `<off>` code that builds one from raw bytes sets to 1), then `<offset> <name> <type>` per field, inherited ones first; it is `""` for other types. User code rarely needs them.

`core.mulHi(u64 a, u64 b) : u64` is the high 64 bits of the 128-bit product `a * b` (one `MUL` on x86-64, `UMULH` on arm64); `core.mulWide(a, b) : (u64, u64)` is the whole product as `(hi, lo)`. `core.addCarry(u64 a, u64 b, u64 carry) : (u64, u64)` is `(a + b + carry, carry out)` and `core.subBorrow(u64 a, u64 b, u64 borrow) : (u64, u64)` is `(a - b - borrow, borrow out)`; the carry or borrow in must be 0 or 1, as the one out is. They are the steps of multi-precision arithmetic in 64-bit limbs, the shape of Go's `bits.Mul64`, `Add64` and `Sub64`. None of them branches, so they take the same time for every input. The backends keep carries in the flags: `mulWide` is one `MUL` on x86-64 (both halves come from RDX:RAX); `addCarry` or `subBorrow` calls whose carry feeds the next (or is added to a value, as in `hi + c`) become `ADD`/`ADC` or `SUB`/`SBB` chains (`ADDS`/`ADCS`, `SUBS`/`SBCS` on arm64), and a carry needed as a value is read out with `SETB` (`CSET`). In the IR they are adds and unsigned compares, which the interpreter runs and the optimizer folds on constants. Shader code cannot use them.

## Wire format (`core.encode` / `net`)

Packed structs are their exact bytes (with their declared byte order).
Otherwise: scalars little-endian at their natural width (`bool` 1 byte,
`char` 4), `str` as u32 length + bytes, `List<T>` as u32 count + elements,
structs/tuples/arrays field by field, enums as an i32 tag + payload, and
`T?` as one byte + value. TCP messages carry a 4-byte little-endian length
prefix; a UDP message is one datagram. With `reliable: true` a datagram
starts with a kind byte (1 a message, 2 an ack) and a u32 sequence number;
with `compress: .lz4` a message body is its u32 length and an LZ4 block.

## Networking

All sockets are non-blocking underneath. Every wait (`accept`, `read`,
`recvFrom`, a TLS handshake, an HTTP response) polls in slices of at most
50 ms, so a task blocked in one stops with `NetError.Cancelled` (or
`HttpError.Cancelled`) soon after it is cancelled, and with `Timeout` after
the timeout it was given (`0.s` means no limit). A `Stream`, `Listener` or
`UdpSocket` is a handle: copies share the socket, which closes on `close()`
or when the last copy goes away. Listening on an empty host (`":8080"`)
takes IPv6 and IPv4 on one socket where the system allows it; addresses of
IPv4 peers on such a socket read as plain IPv4.

`tcpConnect(addr)` tries each address the host resolves to in turn. When
none connects, its error is the last attempt's, from the system's error
number: `Refused(addr)` for a refused connection (`ECONNREFUSED`),
`Timeout` when the system or the given timeout gave up, and otherwise
`Io("connect to <addr>: <reason> (error <n>)")`, such as `network
unreachable`, `no route to host`, `permission denied` or `address not
available`, so a missing route is not mistaken for a closed port.

`Stream.read(ref buf, max)` appends to `buf`. A stream keeps 16 KiB of
read-ahead for `readLine` and small reads; a read of 16 KiB or more with
none of it left goes straight into `buf`, which first grows to hold `max`
more bytes, so a loop that `clear()`s one list and reads into it copies
nothing and allocates nothing after its first read.

**Connected UDP**: `udpConnect("host:port")` (or `connect(addr)` on a
bound socket) fixes the socket's peer: `send(data)` sends to it,
`recv(ref buf, timeout)` takes only its datagrams (the system drops
anyone else's), and when a datagram finds the peer's port closed, the
ICMP port unreachable that comes back fails the next `send` or `recv`
with `NetError.Refused(peer)` (`ECONNREFUSED`; on Windows
`WSAECONNRESET`, which an unconnected socket's `recvFrom` skips), so a
client of a dead server (a DNS upstream) fails at once instead of
waiting out its timeout.

**Datagram batches**: a `Datagrams` holds up to `count` datagrams of up to
`size` bytes (2048 by default) in buffers allocated once.
`UdpSocket.recvBatch(ref batch, timeout)` waits for at least one datagram
and takes as many as are waiting (`recvmmsg` on Linux, a `recvfrom` each
elsewhere); `bytes(i)` and `from(i)` read them without copying.
`sendBatch(batch)` sends everything the batch holds (`push(data, to)`, or
`slot()` to write the next one in place and `commit(n, to)`), with
`sendmmsg` on Linux, where a run of equal-sized datagrams to one address
(the last may be shorter) goes as one UDP GSO message: the kernel cuts it
up after one trip through the network stack. `trySendBatch(batch, from)`
sends from datagram `from` on without waiting, as many as the socket's
buffer takes, and returns how many went (0 when it is full), for code
that cannot wait (std.net.quic sends each connection's round of up to 64
datagrams this way, and its endpoint receives with `recvBatch`). A received batch can be
rewritten in place (`ptr(i)`, `setLen(i, n)`) and sent back to its
senders. Nothing is allocated per datagram, so a packet path (a tunnel, a
game server) moves hundreds of thousands of datagrams a second on one
core. `UdpSocket.setGro(true)` turns on UDP GRO (Linux 5.0 and later; it
returns false and changes nothing elsewhere): the kernel hands over a
run of equal-sized datagrams from one sender as one buffer of up to 64
KiB, which `recvBatch` cuts back into its datagrams (what does not fit
the batch waits for the next call), so a bulk receiver makes far fewer
trips through the kernel. With GRO on, receive with `recvBatch` only:
`recvFrom` would get a run as one datagram.

**HTTP**: the client sends HTTP/1.1 with `Host`, `User-Agent` and
`Content-Length`, reads `Content-Length`, chunked and close-delimited
bodies, skips `1xx` responses, follows `301 302 303 307 308` up to
`maxRedirects` (303, and 301/302 after a POST, become GET; credentials are
dropped across hosts), and can go through an HTTP proxy (`proxy`, with
CONNECT for https). `Client` keeps connections alive and retries once on a
new one when a kept connection turns out closed. The server reads each
request's body (`Content-Length` or chunked, `Expect: 100-continue`, up to
`maxBodyBytes`), answers 400, 404, 405 (with `Allow`), 408 and 413 itself,
keeps connections alive (HTTP/1.1 unless `Connection: close`; HTTP/1.0 with
`keep-alive`; an idle one for `idleTimeout`), serves each connection on its
own task (up to `maxConnections`), and stops when its task is cancelled.
`maxBodyBytes`, `maxConnections`, `idleTimeout` and `readTimeout` set to 0
mean no limit. Route patterns are literal segments, `:name` segments and a
trailing `*`, matched against `req.path`, which is percent-decoded;
`req.rawPath` is the path as sent. Not supported:
HTTP/2 and cookies. A handler that panics gets a 500 (below).

**gzip**: the client sends `Accept-Encoding: gzip` and decodes a gzip (or
`deflate`, which is zlib-wrapped) body, dropping `Content-Encoding` and
`Content-Length` from `fields`; a body that fails to decode is
`HttpError.BadResponse`. A request that sets `Accept-Encoding` itself gets
the body as sent. On the server, `resp.gzipFor(req)` gzips a body of 1 KiB
or more when the request accepts gzip, and adds `Vary: Accept-Encoding`.

**Socket options**: `tcpListen`, `tcpConnect` and `udpBind` take a
`SockOptions`, applied after the socket is made and before it binds,
listens or connects: `reuseAddr` (`SO_REUSEADDR`; by default on for TCP
listeners only, and on Windows `SO_EXCLUSIVEADDRUSE`), `reusePort`
(`SO_REUSEPORT`, Linux and macOS; `NetError.Unsupported` on Windows),
`recvBuffer` and `sendBuffer` (`SO_RCVBUF`, `SO_SNDBUF`), `v6Only`
(`IPV6_V6ONLY`; by default an empty host is dual-stack), `noDelay`
(`TCP_NODELAY`, on by default), `keepAlive` with `keepAliveParams`
(`TCP_KEEPIDLE`, `TCP_KEEPINTVL`, `TCP_KEEPCNT`, in whole seconds),
`linger` (`SO_LINGER`: `0.s` makes close reset the connection), `device`
(only this interface: `SO_BINDTODEVICE` on Linux, which may need
`CAP_NET_RAW`, `IP_BOUND_IF`/`IPV6_BOUND_IF` on macOS, `Unsupported` on
Windows), `bindAddr` (connect from this local address) and `ttl`. A
listener gives the streams it accepts its `noDelay`, `keepAlive` and
`linger`. The same options are read and changed on open sockets
(`s.setLinger(0.s)`, `l.recvBufferSize()`, ...); Linux reports buffer sizes
doubled. `setRawOption(level, name, value)` and `rawOption(level, name)`
reach any other int option by its system numbers. `shutdown(.Write)` is a
half-close: the peer reads the end of the stream and can still answer.

**HTTP server**: requests are read within limits: the request line and
header fields together within `maxHeaderBytes` (64 KiB; 414 for a line
past it, 431 for fields), at most `maxHeaders` fields (100; 431), a body
within `maxBodyBytes` (16 MiB; 413, before reading it when its
`Content-Length` says so). Framing is strict, as RFC 9112 asks against
request smuggling: `Transfer-Encoding` and `Content-Length` together, a
bad or conflicting `Content-Length`, a transfer coding that does not end
in chunked, folded or malformed field lines, a missing `Host` on
HTTP/1.1 and a target that is not `/...`, a URL, `*` (OPTIONS) or
`host:port` (CONNECT) get 400; an `Expect` other than `100-continue`
gets 417, a version other than 1.0 or 1.1 505. Chunk extensions and
trailer fields are read and dropped. Pipelined requests are answered in
order, and a client that half-closes after its request still gets its
answer. Header fields keep their order, case and repeats
(`req.headers.list`, `headers.values(name)`); values sent are kept on one
line (CR and LF become spaces) and fields with bad names are dropped.
Every response gets a `Date` field (`sendDate`).

A route's body is read whole before its handler runs, unless the route
streams it (`stream(method, pattern, handler)`, or `routeWith` with
`{ streamBody: true }`): then `req.reader()` reads it as it arrives
(`read`, `chunk`, `for await c in body.chunks()`, `readAll`, `readText`,
`readLine`, `saveTo(path)`), still within `maxBodyBytes`
(`body.error()` is `HttpError.TooLarge` past it), and `100 Continue` goes
out when the handler first reads, not before. A body the handler leaves
unread is drained after the response if it is short (256 KiB), else the
connection closes (as it does when the client still waits for its 100).

A handler sends a response as it is produced with `req.stream(head,
w => ...)`: `head`'s status and fields, then what the function writes to
the `BodyWriter` (`write`, `writeText`, `flush`), chunked, or with
`Content-Length` when `head` has one, or to the end of the connection for
an HTTP/1.0 client; writes are buffered up to 16 KiB and `flush` sends at
once. `req.sse(sse => ...)` answers with `text/event-stream` and an
`SseWriter` (`send(Event { data, event, id, retry })`, `data`, `event`,
`comment`), flushing each event. `req.upgrade(head, s => ...)` sends
`head` (a 101 with `Upgrade` and `Connection`) and gives the function the
raw `net.Stream`, both ways; `req.tunnel(s => ...)` does the same after a
`200 Connection Established` for a CONNECT (`Server.connect(handler)`,
the target in `req.path`), and `http.pipe(a, b)` copies two streams into
each other until both ends close, passing on half-closes: a CONNECT proxy
or a WebSocket proxy is a few lines. The connection closes when the
function returns. Because lambdas capture by reference, these run their
function before they return, inside the handler. Called outside a server
(`app.handle(req)`), `stream` collects the body into the response.

**Streamed bodies on hand-built messages.** A `Request` or `Response`
carries a body that is read as it is sent in `bodyReader`
(`resp.withReader(r)` sets it on a response), a `BodyReader` from any of:
`BodyReader.ofFile(path)` (`! fs.IoError`; read a piece at a time, its
length the file's size, closed at the end), `ofStream(stream, length)`
(that many bytes of a `net.Stream`, or with -1 all of it), `ofText` and
`ofBytes` (in memory), another message's body (`req.reader()` on a
streamed route, `resp.reader()` after `http.open`), or `http.bodyPipe()`,
whose `BodyWriter` a task writes while the reader is sent:

```bismuth
ref app.get("/archive", req => {
    let (w, r) = http.bodyPipe();
    let t = spawn writeArchive(w);       // writes w, then w.finish()
    t.detach();                          // runs on after the handler returns
    http.Response.new(200, "", "application/zip").withReader(r)
});
ref app.get("/raw/:name", req => match http.BodyReader.ofFile("files/{req.param("name") ?? ""}") {
    Ok(r) => http.Response.new(200, "", "application/octet-stream").withReader(r),
    Err(e) => http.Response.text(404, "no such file\n"),
});
ref app.get("/proxy", req => (await http.open({ url: "http://upstream/x" })).or(http.Response.text(502, "")));
let iso = http.BodyReader.ofFile("big.iso").expect("big.iso");
let put = await http.send({ method: "PUT", url: "http://host/big.iso", bodyReader: iso })?;
```

The server sends a handler's response body as it reads it: with
`Content-Length` when the reader's `sendLength()` knows it (a file, bytes
in memory, a pipe given a length, a body nothing has been read from yet
that came with one; not a body decoded from gzip), or the response's own
`Content-Length` field, else chunked (to the end of the connection for an
HTTP/1.0 client). The first piece is read before the head goes out, so a
body that fails at once is answered with 502; one that fails later ends
the connection, so the client sees it cut short. HEAD and statuses
without a body only close the reader. The client sends a request's
`bodyReader` the same way (`Content-Length` or chunked; not again for a
307 or 308, and dropped, closed, when a 301 to 303 turns the request into
a GET). On a route that streams its body (`Server.stream`, `routeWith`),
`req.bodyReader` is the body as it arrives, so passing the request on to
`http.send`, or answering with `withReader(req.reader())`, streams it
through. In process, `app.handle(req)` gives the handler a hand-built
request's `bodyReader` through `req.reader()` and returns the response's
unread, so one server can dispatch to another's handlers without either
body being held whole. HTTP/3 (`std.net.http3`) sends and answers with
`bodyReader` too.

`bodyPipe(length = -1, capacity = 8)` joins a writer and a reader through
a channel of `capacity` pieces (writes are buffered to 16 KiB pieces, and
wait while the reader is that far behind). `finish()` ends the body; a
writer that fails or is dropped without finishing (its last copy gone;
`abort()` does it at once) cuts the body short, which the reader reports
as `HttpError.Io`; once the reader is closed (a client gone away), the
writer's next write fails with `HttpError.Io`. With a `length`, it is sent
as `Content-Length` and the writer must write exactly that much. Run the
writer in its own task, detached when the reader is returned from a
handler (a task is otherwise cancelled with the scope that spawned it).

`http.statusText(code)` is the reason phrase sent when a response has no
`reason` of its own: the RFC 9110 phrases, with hyper's (Rust's http
crate) for 413 `Payload Too Large`, and 422 `Unprocessable Entity` and 507
`Insufficient Storage` among them.

Routing: `use(mw)` adds middleware, `fn(Request, Next) : Response`, run in
the order added around every request (404s and 405s too) once it is
routed; it calls `next.run(req)` to go on, or answers itself (auth, rate
limits), and may change the request or the response (logging, headers).
`routeWith(method, pattern, { middleware: [...] }, handler)` adds
middleware for one route. `nest("/api", api)` mounts another server's
routes under a prefix, with its middleware inside this server's
(`nest` copies them). A path that matches routes of other methods gets
405 with `Allow` (HEAD listed with GET), OPTIONS gets 204 with `Allow`,
HEAD uses GET routes, a GET without `Upgrade` to a WebSocket route gets
426, and `fallback(handler)` replaces the 404. Queries and forms decode
into structs: `req.queryAs<T>()`, `req.formAs<T>()`,
`http.decodeForm<T>(text)`, fields by name with their defaults when
missing, values parsed as the field's type (decimal numbers, bools as
`true`/`false`/`1`/`0`/`on`/`off`/`yes`/`no`, enums by variant name, a
`List<T>` as comma-separated values, an empty value as null for `T?`),
and `FormError.Missing`, `Duplicate` or `Invalid` otherwise.
`req.readJson<T>()` and `Response.jsonOf(status, value)` use
std.encoding.json. `static("/assets", dir)` serves files (also
`serveDir(req, root)` and `serveFile(req, path)` in a handler): a
`Content-Type` by extension, `Accept-Ranges: bytes`, one byte range per
request (206 with `Content-Range`, 416 past the end; several ranges get
the whole file), `index.html` for a directory, and 404 for paths with
`..`; the file goes out in 64 KiB pieces.

**Panics in handlers**: a request's middleware and handler run on the
connection's task, which catches a panic of theirs (`core.panicHookSet`,
the panic hook form of `core.catchPanics`, with no task per request): a
handler that panics is answered with 500 and its connection closed, the panic is
reported to `Server.onPanic` (`fn(Request, str)`, by default a line on
stderr), the answer is `Server.panicResponse(req)` (by default 500
`internal server error`), and the server goes on serving. Defers and drops pending in the
panicking handler do not run, and a `Mutex` it held stays locked, so
handlers should still return errors as responses where they can.
With `catchPanics: false` a panic in a handler ends the program.

**HTTP client**: `http.open(req)` (and `Client.open`) returns once the
response's head has arrived; `resp.reader()` reads the body as it comes,
gzip or deflate decoded on the fly, and a `Client`'s connection goes back
to its pool when the body has been read to its end (`reader().close()`
gives it up). `send` and the one-line calls read the body whole, up to
`maxResponseBytes` (1 GiB; then `HttpError.TooLarge`). `Request.upload`
(`http.producer(w => ..., length)`) sends a body as it is produced:
chunked, or with `Content-Length` when the length is given; with
`Expect: 100-continue` in the request's headers the body waits for the
server's 100 (or a second) and is not sent if the server answers first.
`connectTimeout` limits connecting, `timeout` each wait, and
`totalTimeout` the whole exchange (each wait is cut to what is left).
`allowRedirect: (from, to) => ...` is asked before each redirect is
followed (a refused one is returned as it is); a streamed upload is not
sent again for a 307 or 308. `postJson(url, value)`,
`getJson<T>(url)` and `resp.readJson<T>()` use std.encoding.json.
`http.upgrade(req)` (`Client.upgrade`) sends a request that switches
protocols, the `Upgrade` and `Connection` fields in its headers, or a
CONNECT with `target: "host:port"`, and returns the response and, after a
101 (or a 2xx to CONNECT), the raw stream. `SseReader.of(resp.reader())`
parses an event stream (`for await e in events`, `lastEventId`).

**WebSocket**: frames follow RFC 6455: a client masks every frame (with
keys from the OS's secure generator) and a server takes only masked ones,
a client only unmasked ones; text is checked to be UTF-8, fragment by
fragment as it arrives; control frames are at most 125 bytes and never
fragmented; reserved bits without an extension, unknown opcodes and
continuation frames out of place are errors. Each error sends the close
code for it (1002 protocol, 1007 bad data, 1009 too big) and ends the
connection; `recv` returns `WsError.Protocol` or `WsError.TooLarge`.
`maxFrame` (16 MiB) and `maxMessage` (64 MiB) limit what is accepted.
Pings are answered with pongs, also between the fragments of a message
being sent (a message's frames stay together; control frames may go
between them); `lastPong` tells when the last pong came and `onPong` (in
`Options` and `ServerOptions`) is called by `recv` with each pong's
payload; `pong` sends an unsolicited one. One `recv` runs at a time;
writers wait for each other by parking, not spinning. Close frames are checked (codes 1000-1003, 1007-1014,
3000-4999; a UTF-8 reason), answered with the same code, and reported as
`WsError.Closed(code, reason)`; `close(code, reason)` starts the closing
handshake and waits up to 2 s for the answer: a `recv` running on another
task sees it, otherwise `close` reads it and keeps the messages that come
before it for the next `recv`. `connect` sends `Host` unless
`Options.headers` has one; `connectOn(stream, url, opts)` runs the
client's handshake over a stream the caller connected. `fragmentSize` (or `sendFragmented`) sends long messages as
several frames. Subprotocols: the client offers `protocols` and rejects
an answer it did not offer; a server picks the first of its own
`protocols` the client offered. With `compress: true` on both sides,
permessage-deflate (RFC 7692) is negotiated without context takeover
either way, so each message is compressed on its own: messages of 64
bytes or more go compressed, and senders that end a message with a final
deflate block or a stored block are read too. A client refuses an
answer with extensions it did not offer.

**Reliable UDP** (`{ proto: .udp, reliable: true }`, both sides): each
message is numbered and acknowledged; the sender resends it every 25 ms
(growing to 250 ms) until it is acked or `timeoutMs` passes, and receivers
drop duplicates and stale messages, so messages arrive once and in order.
`send` and `broadcast` return once acknowledged; a `Conn` gives
`NetError.Timeout` if its peer never acks, a `Server` stops resending to a
silent client. Acks go out while the receiving side reads (`recv`,
`receive`, `messages`, `broadcast`), so keep reading as long as the peer
may still send. One message is in flight per peer at a time.

**lz4** (`{ compress: .lz4 }`, both sides): message bodies travel as their
length (u32) and an LZ4 block, over TCP and UDP.

### TLS

`std.net.tls` is TLS 1.3 (RFC 8446) and TLS 1.2 (RFC 5246), client and
server, written in Bismuth on `std.crypto` and `std.crypto.x509`, as
rustls is in Rust: no OpenSSL or other library is loaded, so a program is
one self-contained binary on Linux, macOS and Windows. A TLS stream is an
ordinary `net.Stream`; its reads and writes wait through net's waits like
any socket's, so they are cancellable and take timeouts.

```bismuth
import std.net;
import std.net.tls;

let s = await net.tls.connect("example.com:443")?;                   // system CAs, SNI, host name check
let s2 = await tls.connect("mesh-7:8443", { verifier: tls.Verifier.pinned([peerSpkiSha256]) })?;
let s3 = await tls.connect("10.0.0.5:443", { caFile: "ca.pem", alpn: ["h2", "http/1.1"] })?;
print("{tls.info(s3).version} {tls.info(s3).alpn}");

let cfg = tls.ServerConfig.new({
    certs: [tls.CertifiedKey.load("a.pem", "a.key")?, tls.CertifiedKey.load("b.pem", "b.key")?],
    alpn: ["http/1.1"],
})?;
let secure = await cfg.accept(await listener.accept()?)?;
cfg.setCertificate(tls.CertifiedKey.fromPem(renewedChain, renewedKey)?);   // later handshakes use it
```

| Item | What |
|---|---|
| Client | `connect(addr, opts)`, `client(stream, host, opts)`; `Options`: `verify` (false: accept anything, for tests), trust anchors from `caFile`, `caDir` and `roots` (an `x509.Roots`; any of them replaces the system's unless `systemRoots: true`), `verifier`, `serverName` (SNI and the name checked; default the host), `alpn`, a client certificate (`certFile` and `keyFile`, or `clientCert`), `minVersion` and `maxVersion` (`.Tls12`, `.Tls13`), `cipherSuites` and `groups` (to restrict them), `timeout` |
| `Verifier` | `standard()` (the default: the chain against the anchors with `x509.verify`, purpose server auth, and the name), `pinned(spkiSha256s)` (the leaf's `Certificate.spkiSha256()` must be one of them; for mesh peers with self-signed certificates), `custom(fn(view PeerInfo) : void ! str)` (an error text rejects the peer), `none()` |
| Server | `serverConfig(certFile, keyFile)`, `serverConfigPem(certPem, keyPem)`, `ServerConfig.new(ServerOptions)`; `ServerOptions`: `certs` (among those whose leaf is valid for the client's SNI name, or all when none is, the first whose key signs in a way the client accepts, so an ECDSA and an RSA certificate for one name serve every client), `resolve` (`fn(view Hello) : CertifiedKey?`, asked first: `Hello` has `serverName`, `sigSchemes`, `alpn`), `alpn` (the server's preference wins; a client offering none of them gets `no_application_protocol`), `clientAuth` (`.None`, `.Optional`, `.Required`) with `clientRoots`, versions, suites and groups; `accept(stream, timeout)`, `update(opts)` and `setCertificate(ck)` swap the settings for later handshakes (copies of the config share them), `options()` |
| `CertifiedKey` | a chain (leaf first) and the leaf's `crypto.PrivateKey` (Ed25519, P-256 or RSA): `new(chain, key)` (checks they match), `fromPem(chain, key)`, `load(certFile, keyFile)` |
| After the handshake | `info(stream) : Info { version, cipherSuite, alpn, serverName, peerCertificates }`; `stream.tlsVersion()`; `keyUpdate(stream, requestPeer)` (TLS 1.3) |
| Misc | `available()` (always true), `version()`; the protocol's numbers (`TLS_AES_128_GCM_SHA256`, `GROUP_X25519`, `SIG_ED25519`, ...) for `cipherSuites` and `groups` |

What is spoken: TLS 1.3 with X25519 and P-256 key shares (a
HelloRetryRequest either way when the peer prefers the other), the suites
TLS_CHACHA20_POLY1305_SHA256, TLS_AES_128_GCM_SHA256 and
TLS_AES_256_GCM_SHA384, the middlebox compatibility mode, KeyUpdate
(also sent by itself before AES-GCM's limits), close_notify and
record_size_limit (RFC 8449, honoured when a client asks); TLS 1.2 with
ECDHE-ECDSA and ECDHE-RSA and AES-GCM or ChaCha20-Poly1305 (RFC 5289 and
7905), the extended master secret (RFC 7627), renegotiation_info (RFC
5746) and the downgrade protection of RFC 8446 4.1.3. Signatures checked:
Ed25519, ECDSA with P-256 and P-384, RSA-PSS (rsae, SHA-256/384/512) and,
for TLS 1.2 and certificates, RSA PKCS#1 v1.5; made: Ed25519, ECDSA P-256
and RSA-PSS (and PKCS#1 v1.5 for TLS 1.2 peers that ask for it).
The same handshake runs over QUIC (RFC 9001) for `std.net.quic`:
`quicClientHandshake` and `quicServerHandshake` on a `Conn` made by
`quicConn` with `QuicHooks` move handshake messages through CRYPTO frames
and hand each traffic secret to the connection, with
quic_transport_parameters, required ALPN, and no middlebox compatibility
mode. Renegotiation is refused with a no_renegotiation warning. Not supported:
session tickets and resumption (a client ignores NewSessionTicket, a
server issues none and skips the 0-RTT data a client sends with another
server's ticket, as RFC 8446 4.2.10 says), 0-RTT, TLS 1.1 and older, RSA key exchange, DHE and
CBC suites, post-handshake client authentication. Failures are
`NetError.Tls(message)`; a failed check sends the alert RFC 8446 asks for
and names it ("certificate verify failed: certificate expired: CN=...",
"the peer sent the alert unknown_ca").

Records are encrypted and decrypted in place in the connection's
buffers, and each flight's handshake messages share records. Secrets (keys, traffic
secrets, ECDHE shares) only go through std.crypto's constant-time code.
ChaCha20-Poly1305 is offered and chosen first because without AES
instructions it is the fastest here. On a recent x86-64, a TLS 1.3
handshake over loopback costs both sides together about 1.9 ms with an
Ed25519 certificate, 2.4 ms with P-256 and 6.8 ms with RSA-2048 (RSA
signing dominates), the same over TLS 1.2; bulk data moves at about
240 MB/s through both ends in one process (encrypting and decrypting).
Tested against RFC 8448's traces (the key schedule, record protection,
CertificateVerify and Finished byte for byte, and the client run against
the traces' server records), against `openssl s_client` and
`openssl s_server`, and over loopback in every combination above.

### QUIC

`std.net.quic` is QUIC version 1 (RFC 9000, RFC 9001, RFC 9002) with
unreliable datagrams (RFC 9221), client and server, written in Bismuth
over net's UDP sockets with `std.net.tls` doing the TLS 1.3 handshake
in CRYPTO frames, as quinn does in Rust: no library is loaded. An
`Endpoint` is a UDP socket and the task that receives its datagrams,
routes them to connections by connection ID and runs their timers; a
client endpoint can make many connections and a server one accepts them.
The endpoint's task takes up to 64 datagrams with one `recvBatch`
(`recvmmsg`, with UDP GRO on Linux: runs arrive coalesced), hands them
to their connections (a run to one connection under one hold of its
lock) and then lets each answer once, and a connection sends a round of up to 64 datagrams with one
`sendmmsg`, as UDP GSO messages on Linux, as quinn does: a bulk transfer
costs a few system calls per 64 KiB, not one per datagram.
Calls wait like a socket's: they are cancellable and take timeouts.

```bismuth
import std.net.quic;
import std.net.tls;

let c = await quic.connect("example.com:4433", { tls: { alpn: ["echo"] } })?;
let s = await c.openStream()?;
await s.writeAll("hello".bytes())?;
await s.finish()?;
let reply = await s.readAll()?;
c.sendDatagram(state.bytes())?;                        // unreliable, never sent twice

let ep = quic.listen(":4433", { tls: tls.ServerConfig.new({ certs: [ck], alpn: ["echo"] })?, retry: true })?;
loop {
    let conn = await ep.accept()?;
    spawn serveConnection(conn);                        // acceptStream, acceptUniStream, recvDatagram ...
}
```

| Item | What |
|---|---|
| Endpoints | `connect(addr, cfg)` (an endpoint of its own), `endpoint(":0")` then `Endpoint.connect` (many connections on one socket), `listen(addr, ServerConfig)`; `ClientConfig { tls, transport, socket }` (`tls` is a `tls.Options`: verification, `alpn` (required), client certificates), `ServerConfig { tls, transport, retry, maxConnections, socket }` (`tls` is a `tls.ServerConfig` whose `alpn` names the protocols); `SocketOptions { recvBuffer, sendBuffer }` (the UDP socket's SO_RCVBUF and SO_SNDBUF in bytes, 4 MiB each by default, 0 for the system's; for `listen`, `connect`'s own endpoint and `endpoint(addr, socket)`; the system may grant less, which is not an error), `Endpoint.recvBufferSize()` `sendBufferSize()` (what it granted) |
| `Connection` | `openStream` `openUniStream` (waiting while the peer's stream limit is reached), `acceptStream` `acceptUniStream`, `sendDatagram` (a full send queue drops its oldest) `sendDatagramWait(data, timeout)` (waits for room instead) `sendDatagramParts(head, body)` (two slices as one datagram, copied once) `recvDatagram` `maxDatagramSize` `datagramSendBufferSpace` `datagramBytesQueued`, `close(code, reason)`, `closed(timeout)` (why it ended; a timeout is the error), `remoteAddr` `alpn` `info` (cipher suite, peer certificates) `stats` (`rtt`, `cwnd`, `bytesInFlight`, `pacingRate`, `mtu`, packets sent, received and lost, `sentPastWindow` (packets that took bytes in flight past the window, probe timeout probes aside: always 0), `keyUpdates`, `datagramsDropped`, `retried`), `keyUpdate()`, `withController(fn(Shared<Controller>))` (the congestion controller, under the connection's lock; changes to its window or rate apply at once), `readController<T>(fn(view T)) : bool` (reads it as a `T`, false when it is not one; nothing else happens, as quinn's `congestion_state()`) |
| `Stream` | `read(buf, max)` (0 at the end), `readAll` `readExact` `readText`, `write` `writeAll` `writeText` (waiting while the stream's send buffer is full), `finish` (FIN), `flush` (wait until the peer acknowledged everything), `reset(code)` (RESET_STREAM), `stopSending(code)` (STOP_SENDING), `setReadTimeout` `setWriteTimeout`, `id`, `isUni` |
| `Options` | `idleTimeout` (30 s; the smaller of both sides' applies), `keepAlive` (PING after this long without sending), `handshakeTimeout` (10 s; a client whose server never answered gives up silently, as at an idle timeout, else it sends CONNECTION_CLOSE), `maxStreamsBidi` and `maxStreamsUni` (100 each: open at once, raised as streams close), `streamWindow` (1 MiB) and `connectionWindow` (8 MiB) of flow control, `sendBuffer` per stream, `maxDatagramSize` (0 turns DATAGRAM frames off), `datagramQueue` (received datagrams kept unread), `datagramSendBuffer` (bytes of datagrams queued to send, 1 MiB, as quinn's), `mtuDiscovery`, `initialRtt` (333 ms: the RTT assumed before the first sample), `congestion` (a `fn(time.Instant now, int mtu) : Shared<Controller>` making each connection's controller; `newReno` by default) |
| Congestion control | `Controller`, a struct to extend as quinn's `Controller` trait is implemented: `onSent(now, bytes, pn)`, `onAck(now, sent, bytes, inFlight, appLimited, rtt)` per packet acknowledged (oldest first within an ACK frame), `onEndAcks(now, inFlight, appLimited, largestAcked)` per ACK frame, `onLost(now, sent, bytes, pn)` per packet lost, `onCongestionEvent(now, sent, persistent, lostBytes)` per round of loss detection, `onMtuUpdate(mtu)`, `window()` (bytes in flight allowed) and `pacingRate()` (bytes per second; 0 paces 5/4 of the window per RTT); times are `time.Instant`s, `rtt` an `Rtt` (`smoothed` `latest` `min` `variance` `sampled`; before the first sample `smoothed`, `latest` and `min` are `initialRtt`, as quinn's). `Options.congestion` makes one per connection; `NewReno` is the default, made by `newReno(now, mtu)` |
| Errors | `QuicError`: `Timeout`, `Cancelled`, `Closed`, `IdleTimeout`, `Refused` (no common version), `ConnectionClosed(code, reason)` and `ApplicationClosed(code, reason)` (the peer's CONNECTION_CLOSE), `Transport(code, reason)` (a protocol error found here), `StreamReset(code)`, `StopSending(code)`, `StreamLimit`, `TooLarge`, `Unsupported`, `Tls(message)`, ...; `message()` names transport errors and TLS alerts |

What is spoken: Initial, Handshake and 1-RTT packets with header
protection, coalesced in datagrams and padded to 1200 bytes when they
carry Initial packets; AES-128-GCM, AES-256-GCM and ChaCha20-Poly1305
(with AES or ChaCha20 header protection); version negotiation (a server
answers versions it does not speak); Retry (`retry: true` makes clients
prove their address with an HMAC-protected token first); transport
parameters, checked both ways against the connection IDs seen; issued
and retired connection IDs with stateless reset tokens (a server
answers packets for connections it no longer has with a stateless
reset); streams of both kinds with stream and connection flow control,
credit given back as the application reads; ACKs (at once in the
handshake, for out-of-order packets and probes, else every second
packet or within a quarter of the RTT), RTT estimation, packet- and
time-threshold loss detection, the probe timeout, pluggable congestion
control (NewReno with persistent congestion by default), and pacing; key updates (by
`keyUpdate()`, by the peer, and every 4 million packets); path MTU
probing from 1200 to 1452 bytes; anti-amplification limits; idle timeout
and keep-alives; CONNECTION_CLOSE with transport and application codes
(and TLS alerts as CRYPTO_ERROR). A server follows a client whose
address changes (NAT rebinding). Not supported: 0-RTT and session
resumption, client-initiated migration, ECN, CUBIC.

Congestion control is a `Controller` each connection gets from
`Options.congestion`, a function rather than a shared object so that
options, configs and servers holding them can go to other tasks (a
controller that needs settings of its own gets them after the
connection is made, through `withController`). It is told of every packet that counts in flight (the
ack-eliciting ones) as it is sent, acknowledged or declared lost, and
asked for the window and the pacing rate before each datagram. The
packets one ACK frame acknowledges reach `onAck` oldest first, in
ascending packet number across all its ranges, as in quinn; `onEndAcks`
follows them, then the frame's RTT sample (so `rtt` in `onAck` is the
estimate before it) and loss detection, whose `onLost` calls also go
oldest first. A path MTU probe is seen as quinn's controllers see one:
it counts in flight but is not passed to `onSent`; when it is
acknowledged, `onMtuUpdate(mtu)` comes first and the probe's own `onAck`
(with `bytes` the new `mtu`) right after it, in packet number order with
the rest of its frame; when it is lost, neither `onLost` nor
`onCongestionEvent` hears of it (its loss says nothing of congestion)
and only the bytes in flight drop. `appLimited` is quinn's: the sender last ran out of data
while the window and the pacer had room; `inFlight` in `onAck` is the
bytes in flight before that packet came off, which NewReno uses to grow
only while at least half its window is in use. A controller's own pacing
rate holds strictly (the default pacer also lets a packet go whenever
nothing is in flight); a window under one datagram counts as one. Its
methods run under the connection's lock, on whichever task sends or
receives, so they must not wait or call the connection.
`withController` is for changing one: after it the connection takes the
window and rate again, restarts the pacer's wait and sends what they
allow. `readController<T>` only reads (a `view T`, and false when the
controller is not a `T`).

```bismuth
import std.time;

struct Flow : quic.Controller {
    int bytes = 64000;
    int rate = 0;
    fn onAck(ref self, time.Instant now, time.Instant sent, int bytes, int inFlight, bool appLimited, view quic.Rtt rtt) {
        // a delay-based estimate from now - sent, rtt.min ...
    }
    fn onCongestionEvent(ref self, time.Instant now, time.Instant sent, bool persistent, int lostBytes) { self.bytes = max(self.bytes / 2, 2400); }
    fn window(view self) : int { self.bytes }
    fn pacingRate(view self) : int { self.rate }
}

fn newFlow(time.Instant now, int mtu) : Shared<quic.Controller> { new Flow {} }

let c = await quic.connect(addr, { tls: { alpn: ["live"] }, transport: { congestion: newFlow, initialRtt: 100.ms, datagramSendBuffer: 256.kb } })?;
c.withController(cc => {
    let f = cc as Flow;
    if f != null {
        var g = f;
        g.rate = 1250000;                               // tuned for this connection
    }
});
let ok = c.readController((view Flow f) => print("window {f.bytes}"));   // a pure read
```

Datagrams to send wait in a queue measured in bytes
(`datagramSendBuffer`); `sendDatagram` makes room by dropping the oldest
(counted in `stats().datagramsDropped`), `sendDatagramWait` waits for
room, and `datagramSendBufferSpace()` says how much there is, so a
sender can match its rate to the path.

Each connection's state is a sans-I/O core (datagrams in, datagrams
out, timers) behind a lock; its handshake runs on a task of its own,
through TLS's QUIC hooks (`std.net.tls`'s `QuicHooks`). Waits sleep on
the connection's lock and condition (`core.Sync`) in slices of 20 ms,
checking for cancellation, as `Chan` does.
Tested against RFC 9001 Appendix A byte for byte (both Initial packets,
the Retry integrity tag, the ChaCha20-Poly1305 short header packet and
its key update), RFC 9000's integer and packet number examples, over
loopback (hundreds of streams through a limit of 100, 4 MB each way,
datagrams, resets, key updates, idle timeout, keep-alive, Retry, version
negotiation, failed handshakes, and a relay that drops every 11th
datagram and reorders others; a controller of the test's own told of
sends, acknowledgements, losses and the MTU and owning the window and
pacing rate, the datagram queue's byte limit, and the initial RTT), and against aioquic both ways when
Python has it (tests/quic/interop.bi).

### HTTP/3 and WebTransport

`std.net.http3` is HTTP/3 (RFC 9114) on `std.net.quic`, with QPACK (RFC
9204) and WebTransport over HTTP/3 as browsers speak it, as the h3 and
wtransport crates are in Rust. Requests and responses are
`std.net.http`'s `Request`, `Response` and `Headers`, and an
`http.Server`'s routes can be served over HTTP/3 unchanged.

```bismuth
import std.net.http;
import std.net.http3;

let c = await http3.connect("example.com")?;                 // port 443; ALPN h3
let r = await c.get("/index.html")?;
var up = await c.open({ method: "PUT", url: "/files/a.bin" })?;   // stream a body
await up.write(chunk)?;
await up.finish()?;
let resp = (ref up.response())?;

var srv = http3.Server.new(handler);              // fn(ref http3.ServerRequest): read, respond, write, finish
var srv2 = http3.Server.routes(app);              // an http.Server
ref srv.webtransport("/live/:room", session => { ... });   // fn(http3.Session)
ref srv.webtransport("/play", serve, req => ticketOk(req) ? 200 : 403);   // refuse per request
spawn srv.listen(":443", { tls: cfg });           // cfg = tls.ServerConfig with alpn ["h3"]
```

| Item | What |
|---|---|
| Client | `connect(addr, ClientConfig { tls, transport, http })`; `Client.send(Request)` (whole bodies), `get`, `post`, `open(Request)` → `Exchange` (`write` `writeText` `finish` for the request body, `response()` for status and fields, `read` `readAll` for the body as it arrives, `cancel`), `webtransport(path, headers)` → `Session`, `connection()`, `close()`; `Request.url` is absolute (`https://...`) or a path on this server |
| Server | `Server.new(fn(ref ServerRequest))` (streaming), `Server.routes(http.Server)` (bodies read whole, then `handle`), `webtransport(pattern, fn(Session), admit)` (`admit`, a `fn(view http.Request) : int`, decides each CONNECT first: 2xx accepts it, 300 to 599 refuses it with that status and the handler is not called; by default every one is accepted), `maxRequests` per connection; `listen(addr, ServerConfig { tls, transport, http, retry, maxConnections })`, `serve(quic.Endpoint, Options)`; a cancelled server sends GOAWAY |
| `ServerRequest` | `head` (an `http.Request`: method, `url` as sent, `path`, `query`, `headers` with `host` from `:authority`, `remote`, `version` "HTTP/3"), `read(buf, max)` `readAll`, `respond(status, headers)`, `write` `writeText` (respond 200 first if nothing was sent), `finish`, `send(Response)` (whole), `stream()` |
| `Session` | `id` `request` (the CONNECT request with route parameters), `connection()` (the `quic.Connection` underneath: stats, congestion controller, datagram queue), `acceptStream` `acceptUniStream` `openStream` `openUniStream` (plain `quic.Stream`s), `sendDatagram` (copied once, into QUIC's send queue behind the session's prefix) `recvDatagram` `maxDatagramSize`, `isClosed`, `close`; the session ends when its handler returns or its CONNECT stream does |
| `Options` | `qpackTableCapacity` (0: the peer may not use QPACK's dynamic table), `maxFieldSectionSize`, `webtransport` (offer it: needed by a client that opens sessions; servers turn it on when they have WebTransport routes), `maxBodyBytes` |
| QPACK | `encodeFields(fields, huffman)` (static table and Huffman only), `Decoder` (`decode(block, streamId)`, `encoderStream(bytes)`: the dynamic table, answering with the decoder stream's instructions), `huffmanEncode` `huffmanDecode`, `putInt` `getInt` |
| Errors | `H3Error`: `QuicError(e)`, `Protocol(code, reason)` (the connection broke HTTP/3), `Message(reason)` (a malformed message; the stream is reset with H3_MESSAGE_ERROR), `Rejected(code)`, `GoAway`, `NotFound` (no WebTransport endpoint), `Refused(status)` (the server refused the WebTransport session with that status), `BadUrl`, `Unsupported` |

HTTP/3: control streams with SETTINGS (unknown and reserved settings,
frame and stream types ignored as RFC 9114 asks), GOAWAY both ways,
QPACK encoder and decoder streams, field validation (lower-case names,
pseudo-header fields first and once, no connection-specific fields),
1xx responses skipped, trailers read and dropped, extended CONNECT (RFC
9220). WebTransport: SETTINGS_ENABLE_WEBTRANSPORT and the later
WEBTRANSPORT_MAX_SESSIONS, H3_DATAGRAM (RFC 9297 and the draft
codepoint), `sec-webtransport-http3-draft` headers, bidirectional
streams (0x41 and the session ID) and unidirectional ones (type 0x54)
both ways, datagrams prefixed with the quarter stream ID; streams that
arrive before their session's CONNECT wait for it briefly. Not
supported: server push (no MAX_PUSH_ID is ever sent), the dynamic table
when encoding, blocked streams, priorities (RFC 9218; the header passes
through), CONNECT tunnels other than WebTransport (501).
Tested against RFC 9204 Appendix B (B.1 encoded byte for byte; B.2 to
B.5 decoded, with the decoder stream's answers), RFC 7541's Huffman and
integer examples, over loopback, and against aioquic's HTTP/3 and
WebTransport both ways.

## ssh

`import std.net.ssh;` is SSH-2 (RFC 4251 to 4254), client and server,
written in Bismuth on `std.crypto` and `std.net`: no OpenSSL or libssh
is loaded. It interoperates with OpenSSH both ways (the tests run it
against OpenSSH 9.6's `sshd`, `ssh` and `sftp`).

```bismuth
import std.net;
import std.net.ssh;

let c = await ssh.connect("build.lan", ssh.Config {})?;      // port 22; the key must be in ~/.ssh/known_hosts
let key = ssh.PrivateKey.load("/home/ryan/.ssh/id_ed25519")?;  // or load(path, passphrase)
let auth = await c.authPublicKey("ryan", key)?;
if !auth.success { return; }                                 // auth.methods: what the server takes
let out = await c.exec("uname -a", 30.s)?;                   // stdout, stderr, status, signal

var ch = await c.openSession()?;                             // a terminal
await ch.requestPty("xterm-256color", 120, 40)?;
await ch.shell()?;
await ch.writeText("ls\n")?;
var screen = List<u8>();
var errs = List<u8>();
while await ch.readAny(ref screen, ref errs)? { /* ... */ }
await ch.resize(100, 30)?;

let fwd = spawn c.forwardLocal(net.tcpListen("127.0.0.1:8080")?, "127.0.0.1", 80);   // ssh -L
let port = await c.requestRemoteForward("127.0.0.1", 0)?;                              // ssh -R
let rfwd = spawn c.serveRemoteForward("127.0.0.1:3000");
```

| Item | What |
|---|---|
| `connect(addr, Config)` | `"host"`, `"host:port"` or `"[v6]:port"`: TCP, version exchange, the first key exchange and the host key check; then authenticate. `connectStream(stream, host, port, cfg)` runs over a stream already connected |
| `Config` | `checkHostKey` (`fn(host, port, PublicKey) : bool`, default `checkKnownHosts`), `algorithms`, `timeout` (15 s, to connect and exchange keys), `keepalive` (send keepalive@openssh.com after this long quiet; 0 s: never) and `keepaliveMax` (unanswered ones before giving up), `version`, `rekeyBytes` (1 GiB), `rekeyTime` (an hour) |
| `Client` | `authNone(user)`, `authPassword(user, password)`, `authPublicKey(user, PrivateKey)`, `authAgent(user, Agent)`, `authKeyboardInteractive(user, fn(name, instruction, List<Prompt>) : List<str>)`, each an `AuthResult { success, methods, partial }`; `openSession()`, `exec(command, timeout) : ExecOutput { stdout, stderr, status, signal }`, `openDirectTcpip(host, port, originHost, originPort)`, `forwardLocal(listener, host, port)`, `requestRemoteForward(addr, port) : int`, `cancelRemoteForward`, `acceptForwarded(timeout) : Forwarded { channel, addr, port, originAddr, originPort }`, `serveRemoteForward(target)`; `hostKey` `serverVersion` `algorithms` (`Negotiated`) `sessionId` `banner` `serverSigAlgorithms` `isAuthenticated` `isClosed` `error` `rekey` `close` |
| `Channel` | requests: `requestPty(term, cols, rows, pxWidth, pxHeight, modes)` `exec` `shell` `subsystem` `setEnv` (want a reply; a refusal is `SshError.Refused`), `resize` `signal` (no reply); input: `read(ref buf, max)` (stdout; 0 at the end) `readStderr` `readAny(ref out, ref err, max) : bool` `readAll` `setReadTimeout`; output: `write` `writeText` `writeStderr` `writeStderrText` `eof` `close`; `wait(timeout) : ExitStatus { status, signal, message }` `exitStatus()` `isEof` `isClosed` `kind` `number`; events: `next(ref buf, timeout, max) : ChannelEvent` (`Data(n)` and `Stderr(n)` with the bytes appended to `buf`, `Eof`, `Exit(ExitStatus)`, `Resize(cols, rows, pxWidth, pxHeight)` and `Signal(name)` on a server's session, `Closed`, in the order they came; the alternative to the reads, `wait` and the polls below, not to be mixed with them); for servers `exit(status)` `exitSignal(name, coreDumped, message)` `ptySize()` `resizes()` `takeSignal()`. A handle: copies share the channel, which closes with the last |
| `PublicKey` | `Ed25519(bytes)`, `EcdsaP256(P256PublicKey)`, `Rsa(RsaPublicKey)`, `Other(algorithm, blob)` (DSA, certificates and security keys, kept so lists of keys read whole; they verify nothing); `decode(blob)`, `parse(text)` (`"type base64 comment"`), `fromCrypto`, `algorithm` `encode` `text(comment)` `fingerprint` (`SHA256:...`, as `ssh-keygen -l`) `equals` `bits` `verify(data, sigBlob)` |
| `PrivateKey` | `Ed25519`, `EcdsaP256`, `Rsa`; `parse(text, passphrase = "")` and `load(path, passphrase = "")` (OpenSSH's own format, also encrypted, and PEM: PKCS#8, SEC1, PKCS#1), `generateEd25519`, `generateP256`, `fromCrypto`; `publicKey` `algorithms` `sign(alg, data) : List<u8>` (a signature blob) `encodeOpenssh(comment)` (unencrypted, as `ssh-keygen -N ""` writes) |
| known_hosts | `parseKnownHosts(text)`, `loadKnownHosts(path)` (a missing file is empty): `KnownHosts.check(host, port, key) : HostStatus` (`Known`, `Unknown`, `Changed(line)`, `Revoked(line)`), `keysFor(host, port)`; `knownHostsLine(host, port, key, hashed)`, `learnKnownHost(path, host, port, key, hashed)`, `defaultKnownHostsPath()`; `checkKnownHosts` (the default check) and `acceptAnyHostKey` (for tests) |
| authorized_keys | `parseAuthorizedKeys(text) : List<AuthorizedKey { options, key, comment }>`, `isAuthorized(keys, key)` |
| `Agent` | `connectAgent(path = "")` (`$SSH_AUTH_SOCK`; on Windows OpenSSH's pipe `\\.\pipe\openssh-ssh-agent`), `identities() : List<AgentIdentity { key, blob, comment }>`, `sign(blob, data, flags)`, `close` |
| `Algorithms` | `kex`, `hostKeys`, `ciphers`, `macs`: preference lists, best first; empty for `defaultKex()`, `defaultHostKeyAlgorithms()`, `defaultCiphers()`, `defaultMacs()` |
| `ServerConfig` | `new(hostKeys)`, then `onNone(fn(user))`, `onPassword(fn(user, password))`, `onPublicKey(fn(user, key))`, `onKeyboardInteractive(name, instruction, prompts, fn(user, answers))` (each method is offered once its callback is set), `onRequest(fn(view SessionRequest) : bool)` (grant or refuse an exec, shell or subsystem request: refused, the client gets a failure and the channel stays open; `SessionRequest` is `Session`'s fields without the channel), `onSession(fn(view Session))`, `onDirectTcpip(fn(host, port))`, `onTcpipForward(fn(addr, port))`; fields `algorithms`, `version`, `banner`, `maxAuthTries` (6), `authTimeout` (120 s), `keepalive`, `keepaliveMax` |
| Serving | `serve(listener, cfg)` (a task per connection, until cancelled), `serveConn(stream, cfg)` (one connection, until it ends); `Session { channel, user, mode, command, pty, term, cols, rows, modes, pxWidth, pxHeight, envNames, envValues }` with `env(name)` |

**Algorithms.** Key exchange: curve25519-sha256 (RFC 8731, and its
older name curve25519-sha256@libssh.org) and ecdh-sha2-nistp256 (RFC
5656), with OpenSSH's strict key exchange (kex-strict-c/s-v00, the fix
for the Terrapin attack: sequence numbers restart at every NEWKEYS and
nothing but key exchange may come before the first). Host keys:
ssh-ed25519 (RFC 8709), ecdsa-sha2-nistp256, rsa-sha2-512 and
rsa-sha2-256 (RFC 8332), and SHA-1 ssh-rsa for old servers. Ciphers:
chacha20-poly1305@openssh.com, aes128-gcm@openssh.com and
aes256-gcm@openssh.com (RFC 5647 as OpenSSH frames it), aes128-ctr,
aes192-ctr and aes256-ctr (RFC 4344) with hmac-sha2-256 or
hmac-sha2-512 (RFC 6668) or hmac-sha1, each also in OpenSSH's
encrypt-then-MAC form (-etm@openssh.com). Each side may start a new key
exchange at any time, and this one does after 1 GiB or 2^28 packets
either way or an hour. Clients send ext-info-c and read RFC 8308's
server-sig-algs: RSA keys then sign user authentication with
rsa-sha2-512 or -256, or with ssh-rsa when the server names neither.
Not supported: Diffie-Hellman groups, the post-quantum hybrids
(sntrup761x25519, mlkem768x25519), compression, AES-CBC and 3DES, DSA
keys, certificates, security keys (sk-*), GSSAPI and host-based
authentication, X11 and agent forwarding.

**Tasks and waits.** A client has one reader task that reads every
packet: it answers key exchanges, keepalives and channel requests and
files data for the channels; `serveConn` is the server's reader itself,
and runs each session's handler on a task of its own. Every other call
blocks its task: the waits re-check every 50 ms, so a cancelled task
stops with `SshError.Cancelled`, and channel reads take a timeout
(`setReadTimeout`). Packets go out whole, one at a time; between a
KEXINIT and its NEWKEYS other writers wait. Each channel offers a 4 MiB
window, topped up once the application has read 1 MiB, and sends at
most 32 KiB per packet, so a reader that stops reading stops its
sender. Callbacks are ordinary lambdas, which capture variables by
reference: keep what they use alive while the client or server runs
(build a server's config in the function that runs it).

**Keys.** `PrivateKey.parse` reads OpenSSH's `openssh-key-v1` format,
encrypted ones too (bcrypt_pbkdf, then aes128/192/256-ctr or
aes128/256-gcm@openssh.com: what `ssh-keygen` writes), and PEM keys as
`std.crypto` reads them. A key that needs a passphrase and gets none is
`SshError.KeyEncrypted`; a wrong passphrase is `SshError.BadPassphrase`.
RSA keys come from files only (std.crypto has no RSA key generation).
`checkHostKey` runs on the connection's reader task during the first key
exchange; to show the user a key it refused (to trust it or not),
record it there, for example in a `Mutex<str>` with `key.fingerprint()`,
and save an accepted one with `learnKnownHost`.
known_hosts lines are matched as `ssh` does: `host` for port 22 and
`[host]:port` otherwise, patterns with `*`, `?` and `!`, hashed names
(`|1|salt|hash`, HMAC-SHA1), `@revoked`; `Changed` means another key of
the same type is listed for the host.

**Errors.** `SshError`: `NetError(e)`, `Protocol(why)`,
`Disconnected(reason, message)` (the peer's SSH_MSG_DISCONNECT),
`Kex(why)` (no common algorithm, or a failed exchange), `HostKey(why)`
(refused, or a signature that does not verify), `Auth(why)`, `Key(why)`,
`KeyEncrypted`, `BadPassphrase`, `OpenFailed(reason, message)`,
`Refused(what)` (a request the peer turned down), `Closed`, `Timeout`,
`Cancelled`; `message()` describes each.

On a recent x86-64, against OpenSSH on the same machine,
chacha20-poly1305 moves 150 to 170 MiB/s sending and 85 to 100 MiB/s
receiving; AES-GCM and AES-CTR, held back by std.crypto's table-free
AES, 45 to 70 MiB/s.

### sftp

`import std.net.ssh.sftp;` is the SSH File Transfer Protocol, version
3 (draft-ietf-secsh-filexfer-02, what OpenSSH speaks), over a session's
"sftp" subsystem.

```bismuth
import std.net.ssh.sftp;

let fs = await sftp.open(client)?;                  // an authenticated ssh.Client
let data = await fs.read("/var/log/syslog")?;        // whole files, 64 requests in flight
await fs.write("notes.txt", "hello".bytes())?;
for e in await fs.readDir(".")? { print("{e.name} {e.attrs.size} {e.attrs.isDir()}"); }
var f = await fs.openFile("big.iso", sftp.READ)?;
var buf = List<u8>();
let n = await f.readAt(1048576, ref buf, 65536)?;    // or read, write, writeAt, readAll
await f.close()?;

// a server's session handler
if s.mode == "subsystem" && s.command == "sftp" { await sftp.serve(s.channel, "/srv/files")?; }
```

| Item | What |
|---|---|
| `open(client)`, `openChannel(channel)` | start SFTP: a new session with the subsystem, or a channel whose subsystem already runs |
| `Sftp` | `read(path)` `readText` `write(path, data, mode)` `writeText`, `openFile(path, flags, attrs) : File`, `create(path, mode)`, `stat` `lstat` `exists` `setstat(path, Attrs)`, `readDir(path) : List<DirEntry { name, longname, attrs }>`, `mkdir` `rmdir` `remove` `rename` `posixRename` (OpenSSH's posix-rename@openssh.com, else rename), `realpath` `readlink` `symlink(target, link)`, `version` `hasExtension` `close`. A handle: copies share the session, and tasks may use one at once |
| `File` | `read(ref buf, max)` and `write(data)` at the position (shared by copies), `position` `seek`, `readAt(offset, ref buf, max)` `writeAt(offset, data)`, `readAll` (pipelined), `stat` `setstat` `sync` (fsync@openssh.com) `close` |
| `Attrs` | `flags` (`ATTR_SIZE` `ATTR_UIDGID` `ATTR_PERMISSIONS` `ATTR_ACMODTIME`), `size` `uid` `gid` `permissions` (with the file type bits) `atime` `mtime`; `isDir` `isFile` `isSymlink`; `withSize` `withPermissions` `withTimes` for `setstat`; `modeString(mode)` is `ls -l`'s `drwxr-xr-x` |
| Flags | `READ` `WRITE` `APPEND` `CREATE` `TRUNCATE` `EXCLUSIVE`; status codes `OK` `EOF` `NO_SUCH_FILE` `PERMISSION_DENIED` `FAILURE` `BAD_MESSAGE` `OP_UNSUPPORTED` |
| `serve(channel, root, ServeOptions { readOnly, symlinks })` | serves the tree under `root` until the client finishes |

Requests carry ids and answers may come in any order, so whole-file
transfers keep 64 requests of 32 KiB in flight, and short reads are
completed before what follows. There is no reader task: whichever
caller is waiting reads the channel and files other callers' answers.
`symlink` sends its arguments in OpenSSH's order (target, then link),
which is the reverse of the draft's and what servers expect. Errors are
`SftpError`: `SshError(e)`, `Status(code, message)` (`code()`,
`notFound()`) or `Protocol(why)`.

The server confines paths to `root` (`"/"` is root, `..` stops there)
and announces posix-rename@openssh.com and fsync@openssh.com. Symbolic
links already in the tree are followed by the OS, so serve trees whose
links stay inside; clients may make links only with `symlinks: true`.
Its file calls are per OS: `statx` on Linux, the 64-bit-inode `stat` on
macOS, the C runtime on Windows (no owners or links there).

## url

`import std.net.url;` parses URLs as the [WHATWG URL
Standard](https://url.spec.whatwg.org) does, which is what browsers,
`fetch` and Node's `URL` do, and keeps the parts the URL API shows.

```bismuth
var u = url.parse("HTTP://Example.COM:80/a/./b/../c?x=1#top")?;
print(u.href());                        // http://example.com/a/c?x=1#top
print("{u.hostname()} {u.portOrDefault() ?? 0} {u.pathname()}");   // example.com 80 /a/c
let next = u.join("../d?y=2")?;         // http://example.com/d?y=2
print(u.param("x") ?? "-");             // 1
ref u.setParam("q", "a b&c");           // ?x=1&q=a+b%26c
ref u.setHost("api.example.com:8080");
```

Parsing is the standard's state machine, run over the input's bytes:
leading and trailing spaces and C0 controls and every tab and newline are
dropped, the scheme and domains are lowercased, a scheme's default port
is dropped, `.` and `..` segments are resolved, `\` counts as `/` in
special URLs (`http`, `https`, `ws`, `wss`, `ftp`, `file`), and what a
part can't hold is percent-encoded with that part's encode set (UTF-8,
uppercase hex; existing `%XX` escapes are kept). `parse(text)` needs an
absolute URL; `parseWith(text, base)`, `resolve(baseText, text)` and
`u.join(text)` resolve a relative reference (`../x`, `?q`, `//host/p`,
`#f`). A `UrlError` is `MissingScheme` (relative, no base),
`MissingHost`, `BadPort(port)` or `BadHost(host, why)`; `message()` says
it in words. `normalize(text)` is the href of the parsed URL, the
standard's normal form, and `u.equals(v, excludeFragments)` compares two.

Hosts are domains, IPv4 addresses (in every form the standard reads:
`0x7f.1`, `3232235521`, octal parts), bracketed IPv6 addresses (written
as RFC 5952 text), opaque hosts of non-special URLs, or the empty host of
`file:///`; `hostKind()` says which. A domain with non-ASCII characters
goes through UTS #46 (the IDNA mapping table of Unicode 17.0.0, in
`idna_table.bi`: case folding, compatibility forms, ignored and
disallowed code points), NFC (with `std.text.unicode`, so `"cafe\u{301}"`
and `"caf\u{e9}"` are both `xn--caf-dma`; a domain with nothing past
U+02FF is not touched) and Punycode into `xn--` labels; an `xn--` label
beside non-ASCII ones must decode to valid code points in NFC.
`domainToAscii` and `domainToUnicode` do that on their own, and
`punycodeEncode`/`punycodeDecode` are RFC 3492. Not done: the bidi rule
of RFC 5893.

The setters (`setProtocol`, `setHost`, `setPort`, `setPathname`,
`setSearch`, `setHash`, ...) run the parser from the state for their
part, as the URL API's do: a value that doesn't fit is ignored (a special
scheme can't become a non-special one, a URL without a host takes no
port). `setHref` reports a parse failure. Query parameters are the
query read and written as `application/x-www-form-urlencoded` pairs, as
`URLSearchParams` has them: `params()`, `param(name) : str?` (the first),
`paramAll`, `hasParam`, and `setParam` (replaces the first, drops the
rest), `appendParam`, `deleteParam`, `setParams`, `sortParams`, which
rewrite the whole query. `formEncode`/`formDecode` do the same on text,
and `percentEncode(s, set)` encodes with any of the standard's sets
(`EncodeSet.C0Control`, `Fragment`, `Query`, `SpecialQuery`, `Path`,
`Userinfo`, `Component`, `Form`; `Form` writes spaces as `+`).

The tests run every case of the web-platform-tests `urltestdata.json`
(904) and `setters_tests.json` (284), and the `toascii.json` cases that
don't need the bidi rule (85 of 87). `std.net.http` keeps its
own small `Url` for requests.

## Processes

`std.os.process` runs other programs. Arguments are a `List<str>` and
reach the program unchanged: there is no shell, so nothing in them is
expanded or split. A name without a path separator is looked up in `PATH`.

```bismuth
let r = process.run(["git", "status", "--short"])?;
if !r.status.success() { eprint(r.stderr); }

let c = process.start(["sort"], { stdin: .pipe, stdout: .pipe })?;
let out = c.output("b\na\n")?;        // feeds stdin, reads stdout, waits
```

Each of the child's stdin, stdout and stderr is `.inherit` (this
program's own, the default for `start`), `.pipe` or `.discard` (the null
device). `run` pipes stdout and stderr and returns them with the exit
status. `output(input)` writes `input` to a piped stdin while it reads
stdout and stderr together (poll on Linux and macOS, a thread per pipe on
Windows), so a child that writes a lot to both never waits on a full pipe.
`readStdout` and `readStderr` read what has arrived, for streaming;
reading one pipe to its end while the child fills the other can wait
forever, which is what `output` is for. `pollStdout(ms)` and
`pollStderr(ms)` wait at most that many milliseconds (negative: no limit)
for something to read and report whether there is, so one task can read
with a timeout. `closeStdout()` and `closeStderr()` close this side of
that pipe without ending the child: its next write there fails (SIGPIPE,
or EPIPE if it ignores that, on Linux and macOS; `ERROR_NO_DATA` on
Windows), which ends most programs that only write output, as a pipe's
reader going away does, and reads afterwards fail with `NotPiped`. A read
another task has in flight finishes first (when the child next writes or
ends), and the pipe closes as it returns. `wait` closes stdin first; a write another task has in
flight is not cut short (stdin is closed when it ends), and writes after
that fail with `NotPiped`. Starting a child that inherits stdout or stderr
flushes this program's stdout, so their lines come out in order.

`Options.env` changes the child's environment in order: `setEnv(name,
value)`, `removeEnv(name)`, and `clearEnv: true` to start from nothing.
`cwd` sets its working directory. On Windows, names compare without case.

`newSession: true` starts the child in a session of its own (setsid) with
no controlling terminal, and `newProcessGroup: true` in a process group of
its own (setpgid), so a terminal's Ctrl+C or hangup doesn't reach it; on
Windows either is `CREATE_NEW_PROCESS_GROUP`. `creationFlags` adds Windows
`CreateProcess` flags (`process.CREATE_NO_WINDOW`, so the child of a
service or a tray program opens no console window, `DETACHED_PROCESS`,
`CREATE_NEW_CONSOLE`, `CREATE_NEW_PROCESS_GROUP`) and is ignored
elsewhere.

`ExitStatus` has the exit `code`, or on Linux and macOS the `signal` that
ended the process (`code` is then null); `kill()` sends SIGKILL there and
calls `TerminateProcess` with exit code 1 on Windows. `terminate()` sends
SIGTERM, which the child may catch to clean up; Windows has no such
request every program takes, so there it ends the child as `kill` does. A `Child` is a
handle: copies share the process, and when the last goes away its pipes
are closed. The process is not killed then; one never waited for is
reaped if it has already ended.

Errors are `ProcessError`: `NotFound(path)` (the program or the working
directory), `PermissionDenied(path)`, `InvalidInput(reason)` (an empty
argument list, a NUL byte, a bad variable name), `NotPiped(stream)` and
`Io(message)`.

Per OS: on Linux, glibc's `posix_spawnp` (a `clone(CLONE_VM |
CLONE_VFORK)` and `execve`, with exec errors returned to the caller), pipes
from `pipe2(O_CLOEXEC)`, and every descriptor above 2 closed in the child
(glibc 2.34; a working directory needs 2.29). On macOS, `posix_spawnp`
with `POSIX_SPAWN_CLOEXEC_DEFAULT`. The child starts with no signals
blocked and SIGPIPE at its default; this program ignores SIGPIPE once it
pipes a child's stdin, so writing to a child that has exited is an error
rather than the end of the program. On Windows, `CreateProcessW` with the
command line quoted as the C runtime parses it back (`CommandLineToArgvW`
rules: quotes around arguments with spaces, tabs or quotes, `\"` for a
quote, backslashes doubled only before a quote), UTF-16 throughout, and an
explicit handle list, so the child inherits its three handles and nothing
else. `.bat` and `.cmd` files are refused: cmd.exe parses its command line
by other rules, so run `cmd.exe /c` with your own quoting.

## Pseudo-terminals

`std.os.pty` runs a program in a pseudo-terminal, as a terminal emulator
does: the program sees a terminal on its stdin, stdout and stderr
(`isatty` is true, `stty size` reports the size set, shells turn on line
editing, colors and job control), and this side reads what it draws and
writes what is typed.

```bismuth
var opts = pty.Options { size: { cols: 120, rows: 40 }, cwd: "/home/ann" };
ref opts.setEnv("TERM", "xterm-256color");
let p = pty.start(["/bin/bash", "-l"], opts)?;   // or ["cmd.exe", "/d", "/s", "/c", line]
let reader = spawn pump(p.clone());              // a task reading the output
p.write("ls\r")?;                                // typed input: "\r" is Enter
p.resize(100, 30)?;                              // the program gets SIGWINCH
let status = p.wait()?;                          // process.ExitStatus

fn pump(pty.Pty p) {
    var List<u8> buf = [];
    loop {
        ref buf.clear();
        let n = p.read(ref buf).or(0);           // up to 64 KiB a call
        if n == 0 { break; }                     // the terminal is closed
        // ... feed buf to a std.term.vt screen, a WebSocket, ...
    }
}
```

`start(argv, opts)` takes the arguments as `process.start` does (no shell;
a name without a path separator is looked up in `PATH`, the child's when
`opts.env` sets it) and `Options` has `size` (`Size { cols, rows }`,
80x24 by default), `cwd`, `clearEnv` and `env` (`setEnv`, `removeEnv`).
Nothing sets `TERM`: set it to what the reader of the output understands.

`read(ref buf, max)` appends what the terminal has written, waiting until
something arrives, and returns the count: 0 once the terminal is closed,
which is when the child and every program that inherited the terminal
from it have ended (a background job keeps it open), or after `close()`.
In a task the wait parks the task in the scheduler's poller on Linux and
macOS, as socket reads do; on Windows it hands its thread over
(`blocking`). The terminal processes what passes: output line ends come
as `"\r\n"`, input is echoed unless the program turns echo off, `"\x03"`
is Ctrl+C and `"\x04"` (at the start of a line) the end of input.
`readAll()` reads to the end. `write(text)` and `writeBytes(bytes)` send
input; `resize(cols, rows)` changes the size (`size()` reports it);
`pid()` is the child's process id.

A `Pty` is a handle: `clone()` shares it, so one task reads while others
write, resize, kill or wait. `kill()` ends the child (SIGKILL;
`TerminateProcess` on Windows) and works while another task's read waits:
that read returns 0 once the terminal closes. `wait()` waits for the
child (handing its thread over in a task) and `tryWait()` asks without
waiting; both return a `process.ExitStatus`. `close()` closes the
terminal: reads in progress return 0, writes fail with `Closed`, and the
child's session is hung up (SIGHUP) on Linux and macOS, or the
pseudoconsole closed on Windows, which ends the programs attached to it.
When the last handle goes away the terminal is closed; the child is not
waited for (one that has ended is reaped).

Errors are `PtyError`: `Unsupported(reason)` (Windows before 10 1809),
`NotFound(path)` (the program or the working directory),
`PermissionDenied(path)`, `InvalidInput(reason)` (an empty argument list,
a NUL byte, a bad variable name, a size outside 1..32767), `Closed`,
`Cancelled` (the waiting task was cancelled) and `Io(message)`.

Per OS: on Linux and macOS, `posix_openpt`, `grantpt`, `unlockpt` and
the slave's name (`ptsname_r`; `TIOCPTYGNAME` on macOS), the size set with
`TIOCSWINSZ`, and the child started with `fork` and `execve`: it calls
`setsid`, makes the slave its controlling terminal (`TIOCSCTTY`) and its
stdin, stdout and stderr, marks every other descriptor close-on-exec
(`close_range` on Linux 5.11 and later), unblocks and un-ignores signals,
changes directory and execs; a failed `chdir` or exec comes back as the
error of `start`. The master is close-on-exec and non-blocking, and this
side's copy of the slave is closed once the child has it, so the end
arrives (Linux reports it as EIO, which reads as 0). On Windows, a ConPTY
pseudoconsole (`CreatePseudoConsole`, looked up in kernel32 at run time,
`ResizePseudoConsole`, `ClosePseudoConsole`) between two anonymous pipes,
and the child started by `CreateProcessW` through `STARTUPINFOEX` with
`PROC_THREAD_ATTRIBUTE_PSEUDOCONSOLE` (its std handles set invalid, so a
redirected stdout of this program is not inherited). The pseudoconsole
keeps its output pipe open after its programs end, so a thread pool wait
on the child closes it when the child ends; the reader then gets the last
of the output and the end. Its output is VT text: the pseudoconsole
renders a screen and sends it as escape sequences (clearing, cursor moves,
a title) rather than passing the program's bytes through.

## D-Bus

`std.os.dbus` talks to the session and system buses (and to peers) in
Bismuth from the socket up, with no libdbus: what Rust programs get from
rustbus or zbus.

```bismuth
import std.os.dbus;

let bus = await dbus.session()?;
let v = await bus.getProperty("org.freedesktop.UPower", "/org/freedesktop/UPower/devices/DisplayDevice",
    "org.freedesktop.UPower.Device", "Percentage")?;
print("battery {v.asF64() ?? 0.0:.0}%");

for m in await bus.subscribe({ iface: "org.freedesktop.login1.Manager", member: "PrepareForSleep" })? {
    print("sleeping: {m.body[0].asBool() ?? false}");
}
```

**Connecting.** `session()` reads `DBUS_SESSION_BUS_ADDRESS`, else uses
`$XDG_RUNTIME_DIR/bus` or `/run/user/UID/bus`; `system()` reads
`DBUS_SYSTEM_BUS_ADDRESS`, else `/var/run/dbus/system_bus_socket`;
`connect(address)` takes any address list (`unix:path=...`,
`unix:abstract=...`, several separated by `;`, values `%`-escaped).
The connection authenticates with SASL EXTERNAL as the process's uid,
negotiates Unix descriptor passing (`canPassFds()`), and says `Hello`
(`uniqueName()`); `{ hello: false }` is for peer-to-peer connections.
Other transports (`tcp:`, `launchd:`, `autolaunch:`) are
`DbusError.Unsupported`, and so is connecting on Windows and macOS; the
message and wire code works everywhere.

**Values.** A `Value` is one value of any type: `Byte` `Bool` `Int16`
`Uint16` `Int32` `Uint32` `Int64` `Uint64` `Double` `Str` `ObjectPath`
`Signature` `Fd`, `Bytes` (an `ay`, kept as bytes), `Array(elem, items)`
(the element signature, so empty arrays have a type), `Dict(key, val,
entries)`, `Struct(fields)` and `Variant`. `sig()` is its signature, so a
body is just a `List<Value>`. `variant(v)`, `strs(list)`, `paths`,
`array`, `dict` and `VarDict` (the `a{sv}` options most APIs take) build
them; `asStr` `asBool` `asU32` `asInt` `asF64` `asFd` `asStrs` `asBytes`
`items` `entries` `get(key)` read them and look through variants, so a
property or an `a{sv}` entry needs no unwrapping. `str()` prints a value
as gdbus does (`{'a': <uint16 5>}`); interpolating a `Value` directly
prints the enum's generated form.

**Calls and messages.** `call(msg, timeout = 25.s)` sends a method call
and waits for its reply; an error reply is `DbusError.Remote(name,
message)`. `callMethod(dest, path, iface, member, args)` returns the
reply's body. `send` sends without waiting and returns the serial for
`waitReply`. Messages are built with `methodCall`, `signal`,
`methodReturn` and `errorReply`, or as a `Message { ... }` literal (flags
`NO_REPLY_EXPECTED`, `NO_AUTO_START`, `ALLOW_INTERACTIVE_AUTHORIZATION`);
names, paths and the body's types are checked before anything is sent.
A `Conn` is for one task at a time: while it waits for a reply it keeps
the other messages it reads (signals, calls to this side) for `receive`,
`waitSignal` and `subscribe`, in order.

**Signals.** `addMatch(Match { kind, sender, iface, member, path,
pathNamespace, destination, args })` asks the bus for the messages a rule
matches (`Match.rule()` is its text, `type='signal',interface=...`);
`waitSignal(rule)` returns the next match, and `subscribe(rule)` adds it
and gives a `for` loop over the matches, which ends when the connection
closes (`err()` says why). `Match.matches(msg)` applies a rule here (a
well-known sender name is not compared: messages carry unique names).

**Descriptors.** `Fd` values travel as `SCM_RIGHTS` control messages
beside the message (`h` on the wire is an index into them). A received
message's descriptors are in `fds`, and its `Fd` values name them; they
are the receiver's to close (`closeFds()`). This is what the
xdg-desktop-portal ScreenCast API needs: `OpenPipeWireRemote` returns the
PipeWire socket as an `h`. [`examples/screencast`](../examples/screencast/main.bi)
runs the whole flow (CreateSession, SelectSources, Start,
OpenPipeWireRemote), listening for each request's `Response` signal on
the request path built from the unique name and a `handle_token`.

**Properties.** `getProperty(dest, path, iface, name)` (the value
without its variant), `setProperty(..., value)` and
`getAllProperties(dest, path, iface) : Map<str, Value>`. Bus helpers:
`requestName(name, flags)` (`NAME_ALLOW_REPLACEMENT`,
`NAME_REPLACE_EXISTING`, `NAME_DO_NOT_QUEUE`; answers `NAME_PRIMARY_OWNER`
...), `releaseName`, `nameOwner`, `listNames`.

**The wire.** Messages are written little-endian and read in either byte
order, with the specification's checks: alignment and zero padding,
UTF-8 strings without NUL, object paths, signatures (at most 255 bytes,
32 nested arrays and 32 structs, dict entries only in arrays with basic
keys), booleans 0 or 1, array lengths that end on an element (at most 64
MiB), variants nested at most 64 deep, messages at most 128 MiB, the
header fields each message type needs. `marshal(values)` and
`unmarshal(bytes, sig, big)` do bodies alone; `Message.encode(serial)`
and `decodeMessage(bytes, fds)` whole messages. `tests/run/dbus_wire.bi`
checks them against messages captured from libdbus and GDBus.

Errors are `DbusError`: `Io`, `Address`, `Auth`, `Protocol` (malformed
input), `Marshal` (values that don't fit their signature, bad names),
`Remote(name, message)`, `Timeout`, `Closed`, `Cancelled` and
`Unsupported`. Waits are `net.waitFd`'s: in slices, so a cancelled task
stops waiting, and `0.s` is no time limit.

## X11 capture and input

`std.window.x11` reads the screen of an X server and drives its pointer
and keyboard, as a screen-sharing host does (what Rust programs get from
x11rb's shm, xtest and xfixes):

```bismuth
import std.window.x11;

let d = x11.open()?;                  // $DISPLAY; x11.open({ display: ":1" })
var frame = x11.Frame {};
loop {
    d.capture(0, 0, d.width(), d.height(), ref frame)?;   // RGBA, rows top to bottom
    encodeAndSend(frame.pixels);
    // ... input from the viewer:
    d.moveTo(x, y)?;
    d.button(x11.BUTTON_LEFT, true)?;
    d.typeKeysym(x11.keysymOf('a'))?;
}
```

**Capture.** With MIT-SHM the server copies the rectangle into a System
V shared memory segment (made once at the screen's size, `shmget` and
`shmat`, removed with the last process that has it), one round trip and
no pixels through the socket; without it (a remote display, no
libxcb-shm, or `{ shm: false }`) `GetImage` sends them through the
socket. The pixels are converted to RGBA, or BGRA (`.Bgra`, a plain copy
on the usual layout), with alpha 255; 16-bit (5-6-5) and other TrueColor
layouts are converted through their channel masks. Capturing into the
same `Frame` at the same size reuses its memory. A rectangle outside the
screen is `X11Error.BadRect`; `refreshSize()` reads the size again after
a mode change.

**Input.** `moveTo(x, y)` and `moveBy(dx, dy)` move the pointer,
`button(n, down)`, `click(n)` and `scroll(dx, dy)` (buttons 4 to 7) press
buttons, `key(keycode, down)` presses keys by keycode (evdev code + 8 on
Linux servers), and `keysym(sym, down)` and `typeKeysym(sym)` by keysym,
looked up in the server's keyboard map (`keycodeOf`; `refreshKeymap()`
after a layout change). `keysymOf(char)` is a character's keysym, and
`XK_RETURN`, `XK_LEFT`, `XK_SHIFT_L` and the like name the rest. A keysym
no key has is an error: nothing remaps a spare keycode. `pointer()` and
`isKeyDown(keycode)` ask the server what it sees. This is XTEST's
FakeInput, so it needs the extension (`hasXtest()`).

**Cursor.** `cursor()` is the pointer's image from XFIXES (`hasXfixes()`):
its position, hot spot, size, a serial that changes with the image, and
straight-alpha RGBA pixels, for drawing the cursor over frames that leave
it out.

libxcb.so.1, libxcb-shm.so.0, libxcb-xtest.so.0 and libxcb-xfixes.so.0
are loaded at run time; without libxcb `open` is
`X11Error.NoDisplay`, and a missing extension is
`X11Error.NoExtension(name)` when it is used. `connection()` is the
`xcb_connection_t*` for calls of your own. On Wayland this works through
XWayland only for X clients' windows; capturing a Wayland desktop goes
through the ScreenCast portal ([D-Bus](#d-bus)). Windows and macOS have
no X server: `open` is `X11Error.Unsupported` there.
`tests/x11/capture.bi` runs all of it on Xvfb.

## unix

`std.os.unix` holds the Linux and macOS system calls that `std.os` leaves
out, typed: what a server would otherwise take from Rust's `libc` and
`nix`. Each struct layout and number that differs between the two systems
(and their errno, signal and ioctl encodings) is in a per-OS file, so the
same code builds for both. A failed call is a `SysError { call, code }`:
the call (`"ioctl(TUNSETIFF)"`, `"statvfs /data"`) and errno, with
`message()` (strerror), `isPermission()`, `isNotFound()` and
`wouldBlock()`; it prints as `statvfs /x: No such file or directory
(errno 2)`.

```bismuth
import std.os.unix;

let n = unix.raiseNofileLimit()?;       // soft RLIMIT_NOFILE up to the hard limit
let st = unix.statvfs("/var/lib/phosphorus")?;
print("{st.available} of {st.total} bytes free, {n} descriptors");
let si = unix.sysinfo()?;               // uptime, load, memory, swap, processes
```

**Processes and users.** `setsid`, `getsid`, `setpgid`, `getpgid`,
`kill`, `killpg` and `processExists(pid)` (true for a live process this one
may not signal); `pid`, `ppid`, the `SIG*` numbers. `uid`, `euid`, `gid`,
`egid`, `setuid` and `setgid` (drop privileges with `setgid` first), and
`userById(uid)` / `userByName(name) : User?` (`name`, `uid`, `gid`,
`gecos`, `home`, `shell`) through `getpwuid_r`, so directory services
work too; `userName(uid)` is the common case.

**Limits and the machine.** `getrlimit(Resource)` and `setrlimit` take a
`Resource` (`.Nofile`, `.Core`, `.Stack`, ...; the numbers differ per OS)
and a `Limit { soft, hard }` where `null` is unlimited.
`raiseNofileLimit()` raises the soft limit on descriptors to the hard
one, capped at what the kernel allows a process (`fs.nr_open` on Linux,
`kern.maxfilesperproc` on macOS, where `setrlimit` refuses more), and
returns it. `uname()`, `statvfs(path) : FsStats` (`total`, `free`,
`available` to unprivileged users, in bytes; `statfs` on macOS, whose
`statvfs` counts in 32 bits), `sysinfo() : SysInfo` (`sysinfo(2)` on
Linux, `sysctl` on macOS), `loadavg()` and `uptime()`.

**ioctl.** `ioctl(fd, request, *u8)` is `unsafe`: what the pointer must
point to depends on the request. `ioctlWith<T>(fd, request, ref T)` is safe
for requests that encode their argument's size: it refuses (EINVAL) unless
that size is `sizeof(T)`. `io`, `ior`, `iow`, `iowr` and `ioc` build
request codes as C's `_IO*` macros do, with the target's encoding (two
direction bits and 14 size bits on Linux, BSD's three and 13 on macOS),
and `iocDir`/`iocSize` take one apart: `unix.iow(int('T'), 202, 4)` is
`TUNSETIFF`, `0x400454ca`.

**TUN devices.** `openTun(TunOptions)` creates an interface whose packets
the program reads and writes: on Linux through `/dev/net/tun` and
`TUNSETIFF` (TUN or TAP, `IFF_NO_PI` unless `packetInfo`, optionally
`IFF_MULTI_QUEUE`), on macOS by connecting a `PF_SYSTEM` socket to
`com.apple.net.utun_control` (`CTLIOCGINFO`; TUN only, `utunN` names). On
macOS each packet carries a 4-byte address family, which `read` strips
and `write` adds (from the IP version) with `readv`/`writev`, so code sees
bare IP packets on both systems. `read(ref buf)` puts one packet in `buf`,
reusing its storage; `setMtu`, `up`, `down` and `setAddress(ip, prefix)`
configure the interface with `SIOCSIF*` ioctls on an `AF_INET` datagram
socket (on a utun, `SIOCAIFADDR` with the address as its point-to-point
destination), and the same calls exist for any interface by name
(`interfaceMtu("lo")`). It needs root or `CAP_NET_ADMIN`: without it
`openTun` fails with `isPermission()`, and where the kernel has no TUN
driver with `isNotFound()`. Linux adds `setPersist`, `setOwner` and
`setGroup`. IPv4 addresses only, for now.

**Watching files.** `watcher()` gives a `Watcher`; `add(path)` returns
a watch id and `read(timeoutMs)` the changes since the last read (empty
on a timeout), each a `Change { watch, path, name, kind, isDir }` with
`kind` one of `Created`, `Modified`, `Attrib`, `Removed`, `Renamed` and
`Overflow`. On Linux it is inotify: changes inside a watched directory
name the entry. On macOS it is a kqueue with an `EVFILT_VNODE` filter per
path, each on an `O_EVTONLY` descriptor, which reports only that the
directory changed (`Modified`, name `""`). Neither recurses. Linux's raw
interface is there too: `inotifyInit(nonBlocking)`, `addWatch(path,
mask)` with the `IN_*` bits, `rmWatch`, `read`, and
`parseInotifyEvents(bytes)`, which decodes `struct inotify_event` records.

Every descriptor here (`Tun.fd()`, `Watcher.fd()`, `Inotify.fd()`) is an
ordinary one that `poll(2)` waits on, so an event loop can wait on it
beside sockets; `waitReadable(fd, timeoutMs)` waits on one, and
`setNonBlocking` switches `O_NONBLOCK`.

## windows

`std.os.windows` is the Windows side: what Rust's `windows-sys` gives a
server, typed. Calls fail with `WinError { call, code }`, the code from
`GetLastError` (or the status a registry function returns) with
`message()` from `FormatMessageW` ("Access is denied."), `isNotFound()`
and `isAccessDenied()`; the `ERROR_*` codes that come up are constants.

```bismuth
import std.os.windows;

let k = windows.createKey(.CurrentUser, "Software\\Phosphorus")?;
defer k.close();
k.setStr("Home", "C:\\phosphorus")?;
let port = k.getDword("Port").or(7777);
let mem = windows.memoryStatus()?;
let disk = windows.diskSpace("C:\\")?;
```

**Registry.** `openKey(root, path, access)` and `createKey` (which makes
missing keys on the way) under `Root.CurrentUser`, `.LocalMachine`,
`.ClassesRoot`, `.Users` or `.CurrentConfig`, with `Access.Read`,
`.ReadWrite` or `.All` (always the 64-bit view). A `Key` reads values as a
`Value` (`Str`, `ExpandStr`, `Binary`, `Dword`, `Qword`, `MultiStr`,
`None`, `Other(kind, bytes)`) or by type (`getStr` takes `REG_SZ` and
`REG_EXPAND_SZ` unexpanded, `getDword`, `getQword`, `getBinary`,
`getMultiStr`; another type is error 1629), writes them (`setStr` ...
`setMultiStr`, whose strings cannot be empty), lists `subkeys()` and
`values()`, and deletes with `deleteValue`, `deleteSubKey` and
`deleteSubTree`; `deleteTree(root, path)` removes a key and everything
under it. Value name `""` is the key's default value.

**Processes and jobs.** `openProcess(pid)` opens another process (by
default with the rights to query, wait for and terminate it):
`terminate(code)`, `wait(timeoutMs)`, `exitCode() : int?` (null while it
runs), `isRunning`, `path`. `listProcesses()` snapshots every process
with Toolhelp32 (`pid`, `parentPid`, `threads`, `exe`). `createJob()` makes
a job object; `assignPid(child.pid())` puts a process in it, and
`setLimits(JobLimits { killOnClose: true })` ends every process in it when
its last handle closes, so helpers cannot outlive a server that crashes.
`activeProcesses()` and `terminate(code)` act on the whole job.

**The machine.** `cpuTimes()` (idle, kernel and user time since boot from
`GetSystemTimes`; two samples give `cpuUsage(a, b)`, 0.0 to 1.0),
`memoryStatus()` (`GlobalMemoryStatusEx`, bytes), `diskSpace(dir)`
(`GetDiskFreeSpaceExW`: `total`, `free`, `available` to this user) and
`uptimeMs()`. `adapters()` lists network adapters with
`GetAdaptersAddresses`: GUID `name`, `friendlyName`, `description`, `mac`
(and `macBytes`), `mtu`, `kind` (`IF_TYPE_*`), `up`, link speeds, `luid`,
unicast `addresses` (with prefix lengths) and `gateways`, IPv6 as RFC 5952
text.

**Wintun.** `loadWintun(path)` loads `wintun.dll` at run time, from the
program's directory or System32 only (`LOAD_LIBRARY_SEARCH_*`, against
planted DLLs), and binds its functions by name; a missing DLL is error
126. `createAdapter(name, tunnelType, guid)` (a fixed GUID keeps Windows
from seeing a new network on every run) or `openAdapter(name)` gives a
`WintunAdapter` (`luid`, `close`), and `startSession(capacity)` a
`WintunSession` over rings of a power-of-two size from 128 KiB to 64 MiB:
`receive(ref buf)` takes a packet without waiting (false when none is
there), `read(ref buf, timeoutMs)` waits on the session's read event,
`send(packet)` queues one (error 111 while the ring is full), `end()`
stops. `logToStderr()` routes the driver's log there. Administrator
rights are needed to create an adapter.

**Starting processes.** `startProcess(commandLine, StartOptions)` and
`startAsUser(token, commandLine, StartOptions)` call `CreateProcessW` and
`CreateProcessAsUserW` with a `STARTUPINFOEXW` and a process-thread
attribute list, for what `std.os.process` leaves out: `jobs` puts the
child in jobs before its first instruction (Windows 10 and later),
`parent` gives it another parent process, `inherit` names the only
handles it inherits (with `stdIn`, `stdOut` and `stdErr`, which are added
to it), `desktop` picks the window station and desktop (`startAsUser`
defaults to `winsta0\default`, the interactive one), `env` replaces the
environment (`startAsUser` otherwise gives the user's own, from
`CreateEnvironmentBlock`), and `flags` adds creation flags
(`CREATE_NO_WINDOW`, `CREATE_BREAKAWAY_FROM_JOB`, `CREATE_SUSPENDED`, whose
main thread `resumeThread` starts). The command line is passed as it is;
`process.commandLine(argv)` quotes one. `Started` holds the `Process`, the
thread id and, while suspended, the thread handle. Jobs also have
`processIds()` (`JobObjectBasicProcessIdList`), `contains(process)` and
`inAnyJob(process)` (`IsProcessInJob`), and `JobLimits.breakawayOk` and
`silentBreakawayOk` let children leave a job.

**Tokens, SIDs and sessions.** `processToken(access)`,
`Process.token(access)`, `threadToken()` (null when the thread is not
impersonating) and `sessionUserToken(session)` (`WTSQueryUserToken`: the
user signed in to a session, for a service running as LocalSystem) open
a `Token`: `user()` (a `Sid`), `isSystem`, `sessionId` and
`setSessionId`, `isElevated`, `integrityLevel` (0x2000 medium, 0x3000
high), `linked` (the other half of a UAC split token), `duplicate(primary,
access)`, `enablePrivilege(name)` (`SE_DEBUG_NAME`, `SE_TCB_NAME`, ...;
one the token does not hold is error 1300), `impersonate()` (undone with
`revertToSelf()`), `environment()` (the user's, as a new logon has it)
and `profileDir()`. `enablePrivilege(name)` does it for this process. A
`Sid` prints as `S-1-5-18` (`parseSid` reads that form back; both are
plain Bismuth), with `bytes`, `ptr` (for a call taking a PSID),
`authority`, `subauthorities`, `rid`, `isWellKnown(WIN_LOCAL_SYSTEM_SID,
...)` and `account()` (`LookupAccountSidW`: domain, name, kind).
`sessions()` lists Terminal Services sessions (`id`, `station`, `state`),
`activeConsoleSession()` is the one at the physical console,
`processSession(pid)` a process's, and `sessionUser(id)` who is signed
in. Impersonation, like a thread's desktop, DPI awareness and MMCSS
registration, belongs to an OS thread, and a task may move between
threads at any `await`: set them on a thread of one's own (`thread`
below).

**Desktops, input and windows.** `openInputDesktop()` opens the desktop
that has the input (Default, or Winlogon under the lock and UAC screens),
`openDesktop(name)` one by name and `threadDesktop()` the calling thread's;
a `Desktop` has `name()` and `setThread()` (`SetThreadDesktop`).
`sendInput(inputs)` synthesizes `Input.Mouse(dx, dy, data, flags)` and
`Input.Key(vk, scan, flags)` events (`mouseMove`, `mouseMoveTo` across the
virtual desktop, `mouseWheel`, `keyEvent`, `unicodeKey`; `MOUSEEVENTF_*`,
`KEYEVENTF_*`). `windowPlacement(hwnd)` and `setWindowPlacement` read and
set a `WindowPlacement { flags, showCmd, minX, minY, maxX, maxY, normal }`.
`setProcessDpiAwareness(DpiAwareness)` (`.PerMonitorV2`, ...),
`setThreadDpiAwareness` (returning the context `restoreThreadDpiAwareness`
puts back), `dpiForWindow` and `dpiForSystem` cover DPI.

**Displays.** `queryDisplayConfig(flags)` reads the CCD configuration: a
`DisplayConfig` of `PathInfo`s (source and target adapter `Luid`s and ids,
mode indexes, output technology, rotation, refresh rate, `isActive`) and
`ModeInfo`s (`source()`: width, height, pixel format and desktop position;
`target()`: the video signal; `setSourcePosition`, `setSourceSize`).
`setDisplayConfig(cfg, SDC_APPLY | SDC_USE_SUPPLIED_DISPLAY_CONFIG | ...)`
applies one; `sourceName` gives `\\.\DISPLAY1` and `targetName` the
monitor's EDID name and device path.

**Devices, folders, the recycle bin.** `deviceInterfaces(classGuid)` lists
the interfaces of a device interface class through SetupAPI (`path` to
open with `CreateFileW`, `flags`). `knownFolder(KnownFolder.LocalAppData)`
(and `RoamingAppData`, `ProgramData`, `Profile`, `Desktop`, `Documents`,
`Downloads`, `Programs`, `Startup`, `ProgramFiles`, `System`, `Windows`,
...; `knownFolderById` for any FOLDERID) is `SHGetKnownFolderPath`, for
this user or, with a token, another. `recycleBin(root)` sizes a recycle
bin, `emptyRecycleBin(root)` empties it and `recycle(paths)` moves files
into it.

**Threads.** `thread(ctx, body)` runs `body(ctx)` on a new OS thread
outside the task scheduler and returns an `OsThread` (`join(timeoutMs)`,
`isRunning`, `close`); `body` is a function or a lambda that captures
nothing, and results go back through what `ctx` holds (`Chan`, `Atomic`,
`Mutex`). `mmcss("Games")` registers the calling thread with the
Multimedia Class Scheduler (`setPriority`, `revert`).
`setProcessPowerThrottling(.Off)` keeps the process out of EcoQoS (`.On`
asks for it, `.Auto` lets Windows decide), as `setThreadPowerThrottling`
does for a thread. `SrwLock.new()` is a slim reader/writer lock (`lock`,
`unlock`, `lockShared`, `unlockShared`, `tryLock`, `tryLockShared`,
`with(f)`) that blocks an OS thread rather than a task, so it also works
in callbacks Windows makes on its own threads; copies share it.

**Registry, more.** `openCurrentUser()` opens the `HKEY_CURRENT_USER` of
the user the thread impersonates (`RegOpenCurrentUser`), which the
predefined key does not follow in a service. `getValue(root, path, name,
expand)` and `Key.getValue(sub, name, expand)` read a value below any key
in one `RegGetValueW` call, expanding a `REG_EXPAND_SZ` when asked;
`keyOf(hkey, path)` wraps a key opened elsewhere.

**UTF-16.** `wide(s)` is a `str` as the NUL-terminated UTF-16 that W
functions take (`wide(s).asSlice().data()`), `toUtf16` the same without
the NUL, `fromUtf16`, `fromWide(p, n)` and `fromWideZ(p)` the other way.
Invalid UTF-8 becomes U+FFFD per maximal invalid subsequence (as Rust's
`from_utf8_lossy` does), unpaired surrogates become U+FFFD. These, and
`ipText`, are plain Bismuth and work on every OS.

## Windows services

`std.os.windows.service` runs a program as a Windows service and manages
services through the Service Control Manager, as Rust's
`windows-service` does.

```bismuth
import std.os.windows;
import std.os.windows.service;

fn serve(service.Service s) {
    let server = startServer();
    s.setRunning();
    for c in s.controls() {
        if c == .Stop || c == .Shutdown { break; }
    }
    server.stop();          // STOP_PENDING was reported when Stop arrived
}                           // returning reports STOPPED

fn main() {
    match service.run("phosphorus", serve) {
        Ok(_) => {},
        Err(e) => {
            if e.code != windows.ERROR_FAILED_SERVICE_CONTROLLER_CONNECT { panic("{e}"); }
            runInConsole();  // not started by the SCM
        },
    }
}
```

`run(name, fn, RunOptions)` hands the main thread to
`StartServiceCtrlDispatcherW` and returns once the service has stopped, or
at once with `ERROR_FAILED_SERVICE_CONTROLLER_CONNECT` when the SCM did not
start the program. The SCM calls back through `export c fn`s: the service
function (its `ServiceMain`) on a thread the SCM starts, after
`RegisterServiceCtrlHandlerExW` and a `START_PENDING` report; the control
handler on the main thread, which answers `INTERROGATE` itself, reports
`STOP_PENDING` (with `RunOptions.stopWaitHint`) for stop, shutdown and
preshutdown, `PAUSE_PENDING`/`CONTINUE_PENDING` for pause and continue,
and queues every control as a `Control` (`Stop`, `Shutdown`,
`Preshutdown`, `Pause`, `Continue`, `ParamChange`, `SessionChange(event)`,
`PowerEvent(event)`, `Custom(code)`, `Other`) on the channel
`controls()` returns. `RunOptions` says which controls are accepted (stop
and shutdown by default); none are while a state is pending, so a stop
never overtakes a start. `setRunning`, `setPaused`, `setStartPending` and
`setStopPending` report progress (pending reports count checkpoints),
`setExitCode` and `setServiceExitCode` what `STOPPED` carries, and
`args()` the start parameters. `RunOptions.reportStopPending = false`
leaves the service `RUNNING` when a stop, shutdown or preshutdown arrives
(as `windows-service` programs usually do) until it calls
`setStopPending` or returns. One service per process
(`SERVICE_WIN32_OWN_PROCESS`).

The SCM side: `install(Config { name, displayName, description, path,
args, start, delayedAutoStart, account, password, dependencies,
restartOnFailure, restartDelayMs })` (`path` `""` is this program; the
command line is quoted, so a path with spaces cannot be hijacked; needs
administrator rights), `uninstall(name)`, `exists`, `start(name, args)`,
`stop`, `pause`, `resume`, `sendControl(name, 128..255)`, `query(name) :
Status { state, pid, exitCode, serviceExitCode, checkpoint, waitHint,
controlsAccepted }` and `waitFor(name, State, timeoutMs)`.

## COM and WinRT

`std.interop.com` calls COM objects through their vtables and makes COM
objects in Bismuth; `std.interop.winrt` reaches the Windows Runtime, which
is COM underneath, as Rust's `windows` crate does. A method is an `extern
c fn` type (`extern com fn` when it returns a struct) read from a vtable
slot, its first parameter the object; IUnknown is slots 0 to 2, and
IInspectable, which every WinRT interface starts with, adds GetIids (3),
GetRuntimeClassName (4) and GetTrustLevel (5), so a WinRT interface's own
methods start at slot 6. Slots come from the SDK headers or windows-rs.

```bismuth
import std.interop.com;
import std.interop.winrt;

type CreateUri = extern c fn(*u8, int, **u8) : i32;      // IUriRuntimeClassFactory::CreateUri
type GetHString = extern c fn(*u8, *int) : i32;

fn host(str text) : str ! com.HrError unsafe {
    winrt.init()?;
    let f = winrt.factory("Windows.Foundation.Uri", com.parseGuid("44a9796f-723e-4fdf-a218-033e75b0c084").expect("iid"))?;
    defer com.release(f);
    var uri = mem.null<u8>();
    let CreateUri create = com.method(f, 6);
    com.check(create(f, winrt.HString.new(text).raw(), mem.addr(uri)), "CreateUri")?;
    defer com.release(uri);
    var int h = 0;
    let GetHString getHost = com.method(uri, 11);
    com.check(getHost(uri, mem.addr(h)), "get_Host")?;
    winrt.takeHString(h)
}
```

**GUIDs.** `Guid` is a GUID as COM lays it out. `==` compares two; it
prints as `{6B29FC40-CA47-1067-B31D-00DD010662DA}` (`str()`), `{g:x}` and
`lower()` in lowercase, and `parseGuid` reads either case with or without
braces. `bytes()` and `guidFromBytes` are its 16 bytes in memory order.
`GUID_NULL`, `IID_IUNKNOWN`, `IID_IAGILEOBJECT`, `IID_IINSPECTABLE`,
`IID_IMARSHAL` and `IID_ICLASSFACTORY` are constants; `newGuid()` makes a
random one on Windows.

**References.** `addRef`, `release` (both ignore null), `releaseCount` (the
count Release reports), `query(obj, iid)` (the interface or null),
`queryHr` (with the HRESULT) and `cast(obj, iid) : *u8 ! HrError`. `Ptr`
owns one reference: `Ptr.attach(p)` takes over an out parameter's,
`Ptr.borrow(p)` AddRefs, copies AddRef, the last copy Releases, and
`raw()`, `method<T>(slot)`, `cast(iid)`, `query(iid)` and `detach()` work
through it.

**HRESULTs.** `failed`, `succeeded`, `hex(hr)` (`0x80004002`),
`hrName(hr)` (`E_NOINTERFACE`, for the codes this module names),
`message(hr)` (`FormatMessageW`'s text, "" when there is none and on
other OSes), `hrText(hr)` (`No such interface supported (0x80004002)`, as
windows-rs prints errors), `fromWin32(code)` (HRESULT_FROM_WIN32) and
`lastErrorHr()`. `check(hr, call)` turns a failure into an `HrError {
call, hr }` for `?`; S_FALSE and other successes pass.

**The runtime and its memory (Windows).** `init(model)` joins the thread
to the multithreaded (default) or an apartment-threaded apartment;
`create(clsid, iid, ref hr, context)` and `newObject(clsid, iid, context)
: Ptr ! HrError` are `CoCreateInstance` with a `CLSCTX_*` context
(`CLSCTX_ALL` by default). `taskAlloc<T>(n)` allocates from the COM task
allocator (`CoTaskMemAlloc`; free with `CoTaskMemFree`), `takeString(p)`
reads and frees a task-allocated UTF-16 string (an out parameter's), and
`taskString(s)` makes one for a callee that frees it. `bstr(s)`,
`fromBstr` and `freeBstr` handle BSTRs.

**VARIANT (Windows).** `Variant` is a `VARIANT`, 24 bytes, passed to
methods as `mem.addr(v)`. It is made by `ofBool`, `ofI4`, `ofU4`, `ofI8`,
`ofU8`, `ofF64`, `ofStr` (a new BSTR) and `ofUnknown` (a new reference),
read by `kind()` (`VT_*`), `asBool`, `asInt` (any integer type), `asF64`,
`asStr` and `asUnknown`, converted by `convert(vt, flags)`
(`VariantChangeType`) and `text()`, deep-copied by `copy()`
(`VariantCopy`) and emptied by `clear()` (`VariantClear`), which a Variant
holding a BSTR or an object needs: copying the struct does not copy what
it holds.

**COM objects in Bismuth.** `implement<T>(iids, methods, state)` makes an
object whose vtable is IUnknown's three methods (provided by this module)
followed by `methods`, addresses of `export c fn`s (`core.fnAddr`) that
take the object first and reach `state` through `state<T>(obj)`.
QueryInterface answers IUnknown, IAgileObject and each of `iids` with the
object itself, AddRef and Release count atomically, and the last Release
drops the state and frees the object; the caller holds the first
reference. The object is agile: it may be called on any thread, so what
the state holds must allow that (`Atomic`, `Mutex`, `Chan`). `refs(obj)`
reads the count, for tests. `delegate<T>(iid, state, invoke)` is the
common case, a WinRT delegate or event handler, whose `Invoke` at slot 3
takes two pointer-sized arguments: `invoke(state, a, b)` is called with
them, borrowed, and returns an HRESULT.

```bismuth
// TypedEventHandler<Direct3D11CaptureFramePool, Object>
let pool = winrt.classSignature("Windows.Graphics.Capture.Direct3D11CaptureFramePool", iidFramePool);
let iid = winrt.pinterfaceIid(winrt.IID_TYPEDEVENTHANDLER, [pool, winrt.SIG_OBJECT]);
let arrived = Atomic<int>(0);
let h = unsafe { com.delegate(iid, arrived, (Atomic<int> n, *u8 sender, *u8 args) => { n.add(1); i32(0) }) };
let token = unsafe { winrt.addHandler(framePool, 8, h)? };      // add_FrameArrived
unsafe { com.release(h); }                                      // the pool holds its own
...
unsafe { winrt.removeHandler(framePool, 9, token)?; }
```

**WinRT.** `winrt.init()` is `RoInitialize` (multithreaded by default).
`HString` owns an HSTRING: `new(s)`, `attach(h)` for an out parameter's,
`raw()` to pass one, `detach()`, `str()`, `len()` (UTF-16 units),
`compare`; copies duplicate it and the last deletes it, and the empty
string is the null HSTRING. `hstringText(h)` reads one the caller does not
own, `takeHString(h)` reads and deletes one it does. `factory(className,
iid)` is `RoGetActivationFactory` (a statics or factory interface, or
`IID_IACTIVATIONFACTORY`), `activate(className)` is `RoActivateInstance`
(a default constructor, giving IInspectable), `className(obj)` and
`iids(obj)` call IInspectable, `close(obj)` calls `IClosable::Close` where
the object has it, `addHandler(obj, slot, handler) : EventToken` and
`removeHandler(obj, slot, token)` call an event's add and remove methods,
and `vectorSize`, `vectorItems<T>` (`GetMany` into a `List<T>` of the
element's ABI type) and `vectorStrings` read an `IVectorView`.

**Parameterized interfaces.** A generic interface instance's IID is derived
from its type signature (a UUID version 5 over SHA-1), which
`pinterfaceIid(generic, args)` computes on any OS:
`pinterfaceIid(IID_IVECTORVIEW, [SIG_STRING])` is IVectorView<String>,
`{2F13C006-A03A-5F69-B090-75A43E33423E}`. Arguments are signatures:
`SIG_I32`, `SIG_STRING`, `SIG_OBJECT` (IInspectable) and the other
fundamental types, `interfaceSignature(iid)`, `classSignature(name,
defaultInterface)`, `structSignature(name, fieldSignatures)`,
`enumSignature(name, flags)`, `delegateSignature(iid)` and, for a nested
instance, `pinterfaceSignature`. The generic IIDs are constants:
`IID_IITERABLE`, `IID_IITERATOR`, `IID_IVECTORVIEW`, `IID_IVECTOR`,
`IID_IMAP`, `IID_IMAPVIEW`, `IID_IOBSERVABLEMAP`, `IID_IOBSERVABLEVECTOR`,
`IID_MAPCHANGEDEVENTHANDLER`, `IID_VECTORCHANGEDEVENTHANDLER`,
`IID_IMAPCHANGEDEVENTARGS`, `IID_TYPEDEVENTHANDLER`, `IID_EVENTHANDLER`,
`IID_IREFERENCE`, `IID_IASYNCOPERATION`,
`IID_ASYNCOPERATIONCOMPLETEDHANDLER`; also `IID_ICLOSABLE`,
`IID_IASYNCINFO`, `IID_IASYNCACTION`, `IID_ASYNCACTIONCOMPLETEDHANDLER` and
`IID_IACTIVATIONFACTORY`.

Not covered: most of the SDK's own interfaces (Media Foundation,
Windows.Graphics.Capture, the shell's) are not declared here; a program
declares the slots it calls. Direct3D 11 and DXGI are declared in
[std.os.windows.d3d11 and std.os.windows.dxgi](#direct3d-11-and-dxgi).
Awaiting an `IAsyncOperation` is done by hand, with a Completed
`delegate`.

## Direct3D 11 and DXGI

`std.os.windows.dxgi` and `std.os.windows.d3d11` (Windows only) declare
Direct3D 11 and DXGI over `std.interop.com`: the structures, constants and
interface IDs of `d3d11.h`, `d3d11_1.h`, `d3d11_4.h`, `dxgi.h` to
`dxgi1_5.h` and `dxgiformat.h`, and a typed wrapper for each interface a
program needs to make a device, fill and read resources, draw and
dispatch, run the video processor, and enumerate adapters and outputs and
duplicate the desktop. `std.gfx`'s Direct3D 11 backend (and its Direct3D
12 backend, for DXGI and the layouts the two share) is built on the same
declarations. `d3d11.dll` and `dxgi.dll` are loaded at run time, so a
program importing them starts on any Windows, and `createDevice` and
`createFactory1` fail with an `HrError` (`ERROR_MOD_NOT_FOUND` as an
HRESULT) where they are missing.

```bismuth
import std.interop.com;
import std.os.windows.d3d11;
import std.os.windows.dxgi;

fn readBack() : List<u8> ! com.HrError {
    // the hardware driver, else WARP, the software rasterizer every Windows has
    let flags = d3d11.CREATE_DEVICE_BGRA_SUPPORT;
    let made = match d3d11.createDevice(d3d11.DRIVER_TYPE_HARDWARE, flags) {
        Ok(m) => m,
        Err(_) => d3d11.createDevice(d3d11.DRIVER_TYPE_WARP, flags)?,
    };
    let desc = d3d11.Texture2DDesc {
        width: 64, height: 64, format: u32(dxgi.FORMAT_B8G8R8A8_UNORM),
        usage: u32(d3d11.USAGE_STAGING), cpuAccessFlags: u32(d3d11.CPU_ACCESS_READ),
    };
    let staging = made.device.createTexture2D(desc)?;
    // ... copyResource(staging, rendered) ...
    let m = made.context.map(staging, 0, d3d11.MAP_READ)?;
    var out = List<u8>();
    unsafe {
        for y in 0..64 {
            let row = mem.offset(m.data, y * int(m.rowPitch));
            for i in 0..64 * 4 { ref out.push(row[i]); }
        }
    }
    made.context.unmap(staging, 0);
    out
}

fn displays() : void ! com.HrError {
    let f = dxgi.createFactory1()?;
    for a in f.adapters()? {
        let d = a.getDesc1()?;
        print("{d.name()}: {d.dedicatedVideoMemory / u64(1.mb)} MB{d.isSoftware() ? " (software)" : ""}");
        for o in a.outputs()? {
            let od = o.getDesc()?;
            print("  {od.name()} at {od.desktopCoordinates.left},{od.desktopCoordinates.top}");
        }
    }
}
```

**Interfaces.** Each interface is a struct holding `ptr`, a `com.Ptr` (an
owned reference: copies AddRef, the last copy Releases), and extends the
struct of the interface it derives from, as `d3d11.Texture2D` extends
`d3d11.Resource`, which extends `d3d11.DeviceChild` and `d3d11.Unknown`;
so a `Texture2D` passes wherever a `view Resource` is asked for. `raw()`
is the interface pointer (still owned), `cast(iid)` is QueryInterface,
and a wrapper is built over a pointer from elsewhere with
`d3d11.Texture2D { ptr: com.Ptr.attach(p) }` (an out parameter's
reference) or `com.Ptr.borrow(p)` (a new one). A wrapper over
`com.Ptr.none()` is a null object, which the `Set` calls take to unbind a
slot. Methods are the COM methods in camelCase (`psSetShaderResources`
for `PSSetShaderResources`), each calling one vtable slot; out parameters
become results, arrays of interfaces `List`s of wrappers, and a failed
HRESULT a `com.HrError` naming the method (`ID3D11Device::CreateTexture2D:
The parameter is incorrect. (0x80070057)`). Calls whose success codes
matter return them: `SwapChain.present` (`STATUS_OCCLUDED`) and
`DeviceContext.getData` (`S_FALSE` until the GPU is done).
`d3d11.Device.dxgiDevice()`, `multithread()` and `videoDevice()`,
`DeviceContext.context1()` and `videoContext()` and
`Resource.dxgiResource()` reach the other interfaces of the same object.

**Making a device.** `createDevice(driverType, flags, featureLevels,
adapter)` is `D3D11CreateDevice`: `DRIVER_TYPE_HARDWARE` or
`DRIVER_TYPE_WARP`, or `DRIVER_TYPE_UNKNOWN` with a `dxgi.Adapter`;
`CREATE_DEVICE_*` flags (`BGRA_SUPPORT`, `VIDEO_SUPPORT`, `DEBUG`); the
feature levels to try, `[]` for Direct3D's list. It returns the device,
its immediate context and the level it got. `createDeviceAndSwapChain`
adds a swapchain for a window described by a `dxgi.SwapChainDesc`.

**Layouts and constants.** The structures are the SDK's with camelCase
fields and its defaults (`Texture2DDesc` one level, one slice, one sample;
`SamplerDesc`, `RasterizerDesc`, `DepthStencilDesc` and
`RenderTargetBlendDesc` D3D11's documented defaults), each checked with
`comptime assert` against the SDK's size. The view descriptions keep
their unions as words, filled by `ShaderResourceViewDesc.buffer(first,
count)` and `.texture2D(format, mostDetailedMip, mipLevels)`,
`RenderTargetViewDesc.texture2D`, `DepthStencilViewDesc.texture2D(format,
mipSlice, multisampled)` and `UnorderedAccessViewDesc.buffer` and
`.texture2D`. Constants drop the `D3D11_` and `DXGI_` prefixes
(`d3d11.BIND_RENDER_TARGET`, `dxgi.FORMAT_NV12`,
`dxgi.ERROR_WAIT_TIMEOUT`); `dxgi.errorName(hr)` names a DXGI error.
Interface IDs are `com.Guid` constants named after the interface
(`d3d11.IID_ID3D11TEXTURE2D`, `dxgi.IID_IDXGIOUTPUT1`).

**Slots.** Every vtable slot a wrapper calls is a constant named after the
interface and method, upper case (`d3d11.ID3D11DEVICECONTEXT_MAP` is 14,
`ID3D11DEVICE_CREATETEXTURE2D` 5, `dxgi.IDXGIOUTPUTDUPLICATION_ACQUIRENEXTFRAME`
8), beside the `extern c fn` type of its signature (`d3d11.MapFn`), for code
that keeps raw interface pointers and calls `com.method`:

```bismuth
let d3d11.MapFn map = com.method(ctx, d3d11.ID3D11DEVICECONTEXT_MAP);
let hr = map(ctx, tex, 0, u32(d3d11.MAP_READ), 0, mem.addr(mapped));
```

They are read from the Windows SDK headers and agree with windows-rs's
generated vtables. Slots no wrapper covers (the geometry, hull and domain
stages, the `Get` side of the context, stream output, predication,
counters, the video decoder and the crypto sessions) are called the same
way, from the headers.

**Desktop duplication.** `Output.duplicateOutput(device)` (with a device
made on the output's adapter: `createDevice(DRIVER_TYPE_UNKNOWN, flags,
[], adapter)`) starts it; `acquireNextFrame(timeoutMs)` waits for a change
and gives `AcquiredFrame { info, resource }`, the desktop image as a
`dxgi.Resource` (cast it to `d3d11.IID_ID3D11TEXTURE2D`), with
`getFrameMoveRects` and `getFrameDirtyRects` (sized by
`info.totalMetadataBufferSize`) and `getFramePointerShape`;
`releaseFrame` gives it back. `DXGI_ERROR_WAIT_TIMEOUT` means nothing
changed and `DXGI_ERROR_ACCESS_LOST` that duplication must start again.
It fails with `E_ACCESSDENIED` where the thread's desktop is not the
interactive one (a service's session 0, the secure desktop) and with
`DXGI_ERROR_UNSUPPORTED` or `DXGI_ERROR_NOT_CURRENTLY_AVAILABLE` where the
display or session can't be duplicated.

Not covered: Direct3D 12 (std.gfx declares what its backend uses,
privately), 1D and 3D textures and arrays of them beyond their view
descriptions' words, Direct2D and DirectWrite, Media Foundation, and the
DXGI 1.6 interfaces (`IDXGIFactory6`, `IDXGIOutput6`).

## time

`import std.time;`. Two clocks, which must not be mixed up:

- `time.now() : Instant` reads the **monotonic** clock. It never goes back
  and its zero is arbitrary; use it to measure elapsed time (`a - b`,
  `i.elapsed()`), for timeouts and frame loops.
- `Timestamp.now()` reads the **wall clock**: Unix time, seconds since
  1970-01-01T00:00:00Z. It names the same instant everywhere, so it is what
  to store, send and show, but the system can step it backwards or
  forwards (manual changes, NTP), so never time anything with it.
  `time.unixMillis()` is the same clock in milliseconds.

The calendar is the proleptic Gregorian one (dates before 1582 counted as
if the Gregorian rules already applied; year 0 is 1 BC) with no leap
seconds, as in ISO 8601 and Unix time. Dates convert to day counts with
Howard Hinnant's `days_from_civil` and `civil_from_days`, so every
conversion is a handful of integer operations with no tables or loops.
Years run from `MIN_YEAR` (-999999) to `MAX_YEAR` (999999). None of the
types below allocates, except a named `Zone`'s shared table and the
strings the text functions return.

| Type | What | Methods |
|---|---|---|
| `Date { year, month, day }` | a calendar date | `new(y, m, d) : Date ! TimeError` (checked), `fromUnixDays(n)`, `fromOrdinal(y, n)`, `fromIsoWeek(y, w, Weekday)`, `unixDays` `isValid` `weekday` `dayOfYear` `isoWeek` (`IsoWeek { year, week }`) `isLeapYear` `daysInMonth` `addDays` `addMonths` `addYears` (clamping the day: 01-31 + 1 month is 02-28 or 02-29) `daysUntil` `at(Time)` `atMidnight` `strftime`, `parse` (`YYYY-MM-DD`), `<` |
| `Time { hour, minute, second, nano }` | a time of day | `new(h, m, s = 0, nano = 0)` (checked), `fromSecondOfDay`, `secondOfDay` `isValid` `strftime`, `parse` (`HH:MM[:SS[.f]]`), `<` |
| `DateTime { date, time }` | both, with no zone | `new(y, mo, d, h = 0, mi = 0, s = 0, nano = 0)`, `fromUnix(secs, nano)` (read in UTC), `year` ... `nano` `weekday` `dayOfYear` `isoWeek` `unixSeconds` `asUtc` `withOffset(secs)` (to a `Timestamp`), `addDays` `addMonths` `addYears`, `+ Duration`, `minus(Duration)`, `a - b : Duration`, `strftime`, `parse`, `<` |
| `Timestamp { secs, nanos }` | a wall-clock instant: Unix seconds, and nanoseconds 0 to 999999999 | `now()`, `fromUnix(secs, nanos)`, `fromUnixMillis`, `fromUnixNanos`, `unixMillis` `unixNanos`, `utc() : DateTime`, `inZone(zone)` `local()` `: ZonedDateTime`, `+ Duration`, `minus(Duration)`, `a - b : Duration` (panics past 292 years), `rfc3339` `strftime`, `parse` (RFC 3339), `<` |
| `Zone` | a time zone; cheap to copy | `Zone.utc()`, `Zone.fixed(secs)`, `Zone.load(name, dir = "")`, `Zone.fromTzif(name, bytes)`, `Zone.posix(rule)`; `name` `isFixed` `offsetAt(ts)` `infoAt(ts) : ZoneOffset { seconds, dst, abbr }` `nextTransition(ts) : Timestamp?` `resolve(DateTime) : LocalResult` `at(DateTime) : ZonedDateTime` `now()` |
| `ZonedDateTime { local, offset, timestamp, zone }` | an instant and its wall-clock reading in a zone | `ZonedDateTime.of(ts, zone)`, `date` `time` `year` ... `nano` `weekday` `abbr` `isDst` `withZone`, `addDays` `addMonths` `addYears` (calendar: the same wall-clock time, resolved again), `+ Duration` `minus` (exact time), `a - b`, `<` (by instant), `rfc3339` `strftime` |
| `Weekday` | `Monday` to `Sunday` | `number` (ISO: 1 to 7), `fromNumber`, `name`, `shortName` |

Also `isLeapYear(y)`, `daysInMonth(y, m)`, `daysInYear(y)`,
`daysFromCivil(y, m, d)`, `civilFromDays(n)`, `monthName(m)`,
`monthShortName(m)`, and `hours(n)`, `days(n)`, `weeks(n)` for the
`Duration`s the prelude has no suffix for. Errors are `TimeError`:
`OutOfRange(what)`, `BadFormat(text, why)`, `UnknownZone(name)`,
`BadZoneData(name)`.

**Zones.** `Zone.load("Europe/Paris")` reads a TZif file (RFC 8536,
versions 1 to 4, including the TZ rule at the end of version 2+ files that
covers times after the table, which "slim" files rely on): from `dir` when
given, else from the directory in `ZONEINFO` when set, else from
`/usr/share/zoneinfo` (and `/usr/lib/zoneinfo`, `/usr/share/lib/zoneinfo`,
`/etc/zoneinfo`). Windows has no zone files, so there `load` needs `dir`
or `ZONEINFO`; `"UTC"` loads anywhere. `localZone()` is `TZ` when it is
set (empty for UTC, a zone name, a path to a TZif file, or a POSIX rule
such as `CET-1CEST,M3.5.0,M10.5.0/3`, with or without a leading `:`), else
`/etc/localtime` on Linux and macOS, and on Windows the system setting from
`GetDynamicTimeZoneInformation`, turned into a rule (its current rule is
applied to every year). Whatever can't be read gives UTC, as C's `tzset`
does. A zone's offsets are seconds east of UTC.

**Gaps and overlaps.** `zone.resolve(local)` maps a wall-clock reading to
instants: `Unique`; `Overlap` when the clocks went back and it happened
twice (`earlier` and `later`); or `Gap` when the clocks jumped over it
(`earlier` is the reading taken with the offset after the jump, `later`
with the one before). `zone.at(local)` takes the later one in a gap (so
02:30 in a 02:00 to 03:00 jump is 03:30) and the earlier one in an
overlap, as RFC 5545 and JavaScript's Temporal do by default. Both look a
day either side of the reading, which is enough for every real zone.

**Text.** `print` shows `Date`, `Time`, `DateTime` and `Timestamp` in
ISO 8601 and RFC 3339 form (`2024-02-29`, `09:05:07.12`,
`2024-02-29T09:05:07.12`, `2024-02-29T14:05:07.12Z`; fractions without
trailing zeros; years outside 0 to 9999 with a sign) and a
`ZonedDateTime` with its offset (`2024-02-29T09:05:07.12-05:00`, `Z` for
UTC, `+HH:MM:SS` for historical offsets that aren't whole minutes).
`parseRfc3339(s) : ZonedDateTime ! TimeError` reads RFC 3339 into a
fixed-offset zone; it also takes `t`, `z` and a space for `T`, `+HHMM` and
`+HH` offsets, a comma before the fraction, and more than nine fraction
digits (the rest are dropped). Leap seconds (`:60`) are refused.

`strftime(pattern)` on every type takes these directives; anything else is
copied as written. `%-d` (and so on) drops the padding. English names only.

| | | | |
|---|---|---|---|
| `%Y` year (4+ digits) | `%y` year % 100 | `%C` century | `%G` ISO week year |
| `%m` month 01-12 | `%d` day 01-31 | `%e` day, space-padded | `%j` day of year 001-366 |
| `%H` hour 00-23 | `%I` hour 01-12 | `%p` AM/PM | `%M` minute |
| `%S` second | `%L` milliseconds | `%f` microseconds | `%N` nanoseconds |
| `%a` `Mon` | `%A` `Monday` | `%b` `%h` `Jan` | `%B` `January` |
| `%u` weekday 1-7, Monday 1 | `%w` weekday 0-6, Sunday 0 | `%V` ISO week 01-53 | `%s` Unix seconds |
| `%F` `%Y-%m-%d` | `%T` `%H:%M:%S` | `%R` `%H:%M` | `%n` `%t` `%%` |
| `%z` `-0500` | `%:z` `-05:00` | `%Z` `EST` | |

`%z`, `%Z` and `%s` know the offset on a `ZonedDateTime` and a `Timestamp`
(UTC); on a `DateTime`, `%z` and `%Z` print nothing and `%s` reads it as
UTC.

## cli

`import std.cli;`. A `Command` declares flags, positional arguments and
subcommands; `parse(args)` reads a `List<str>` (the arguments without the
program's name) into `Matches`, or stops with a `CliError`. Tests pass
their own lists; `parseOs()` reads `os.args()` from index 1, and with
`parseOrExit(args)` prints the help and exits with 0 on `--help`, or
prints the error and usage to stderr and exits with 2.

| Declare (`ref cmd.…`) | Read (`m.…`) | |
|---|---|---|
| `boolFlag(name, short = null, help = "")` | `bool(name)` | false unless given; `--name=true/false` (also `yes/no`, `on/off`, `1/0`) |
| `intFlag(name, short, help, def = 0, required = false, valueName = "N")` | `int(name)` | read with `str.parseInt` |
| `floatFlag(..., f64 def = 0.0, ..., valueName = "X")` | `float(name)` | read with `str.parseFloat` |
| `strFlag(..., str def = "", ..., valueName = "VALUE")` | `str(name)` | |
| `listFlag(name, short, help, required, valueName)` | `list(name)` | repeatable; values in order |
| `arg(name, help = "", required = true)` | `str(name)` | the next positional argument, `""` when absent |
| `restArgs(name, help = "", required = false)` | `list(name)` | the remaining positional arguments |
| `command(sub)` | `command`, `sub() : Matches?` | a subcommand; its name is the first positional argument, and the words after it are its own |

`isSet(name)` reports whether a flag or argument was given. Reading an
undeclared name, or one of another kind, panics. Declaring a flag name
twice, a short form longer than one character, or positional arguments
beside subcommands panics too: those are mistakes in the program.

Accepted forms: `--name value`, `--name=value`, `-n value`, `-nvalue`,
`-n=value`; bundled short bools ending in at most one value flag
(`-vkw 640`); `--` to end the flags; `-` and negative numbers (`-3`, when
no short flag is a digit) as positionals. `--help` and `-h` (unless the
program declares them) stop parsing with a `Help` error whose message is
the generated help: the about text, a usage line, and aligned
`Arguments`, `Options` (with defaults, `(required)` and `(repeatable)`)
and `Commands` sections.

`CliError { kind, message, suggestion, usage, command }`: `kind` is an
`ErrorKind` (`Help`, `UnknownFlag`, `MissingValue`, `BadValue`,
`MissingFlag`, `MissingArg`, `UnexpectedArg`, `UnknownCommand`,
`MissingCommand`); `message` is one line, as `unknown flag --verbos (did
you mean --verbose?)`, and is what printing the error shows;
`suggestion` is the closest declared flag or command within about one edit
per three letters (optimal string alignment distance, so swapped
neighbours count as one edit), or `""`; `usage` and `command` are the
usage line and full name (`git remote add`) of the command that failed.

## sqlite

`import std.db.sqlite;` then `sqlite.open("game.db")?`. The module loads
the system's SQLite when a program first opens a database:
`libsqlite3.so.0` (or `libsqlite3.so`) on Linux, `/usr/lib/libsqlite3.dylib`
on macOS, `winsqlite3.dll` (System32, Windows 10 and 11) or a `sqlite3.dll`
on the search path on Windows; `BISMUTH_LIBSQLITE` names another file.
Without one, `open` returns `SqlError` with code `UNAVAILABLE` (-1) and
`available()` is false.

**Bundled.** A program that calls `sqlite.useBundled()` (before its first
`open`) has SQLite built in and needs no `libsqlite3` where it runs: bic
compiles the amalgamation in `third_party/sqlite` (3.54.1) with the build
machine's C compiler, once (the object is cached, see
[c-interop.md](c-interop.md#static-c-libraries)), and links it statically.
The call is what links it in: programs that don't make it keep loading the
system's library and need no C compiler to build. The bundled SQLite is
built with `SQLITE_THREADSAFE=1`, FTS5, R*Tree, the math functions and
column metadata, without loadable extensions, and `BISMUTH_LIBSQLITE` is
ignored. This is on Linux (x86-64 and arm64), macOS (arm64 and x86-64;
building for macOS from another OS takes zig cc or clang with a macOS SDK)
and Windows (x86-64, where the C compiler is one for `x86_64-windows-gnu`,
such as `zig cc` or mingw-w64's gcc); see
[c-interop.md](c-interop.md#bundled-libraries-compilercbuildbi).

```bismuth
fn main() : void ! sqlite.SqlError {
    sqlite.useBundled();
    let db = sqlite.open("app.db")?;
    print(sqlite.version());   // 3.54.1
}
```

```bismuth
import std.db.sqlite;

struct Score { str name; int points; }

let db = sqlite.open("game.db")?;                  // also ":memory:", and Options
db.exec("create table if not exists scores (name text, points int)")?;
db.exec("insert into scores values (?, ?)", "ryan", 1200)?;
for s in db.query<Score>("select name, points from scores where points > ?", 1000)? {
    print("{s.name} {s.points}");
}
```

| Item | What |
|---|---|
| `open(path, Options)` | `Options { readOnly = false, create = true, uri = false, busyTimeout = 5.s, cacheSize = 32, noMutex = false }`; extended result codes are on. `noMutex: true` opens with `SQLITE_OPEN_NOMUTEX` (see below) |
| `Db` | a handle: copies share the connection, which closes when the last one goes away or at `close()` (for every copy). One thread at a time |
| `exec(sql, a, b, ...)` | runs a statement with up to eight parameters and returns the rows changed; without parameters `sql` may hold several statements |
| `query<T>(sql, ...)` | `List<T>`: each row into a struct, field by column name (exact, else ASCII case-insensitive); extra columns are ignored, a missing one is an error unless the field is `T?` |
| `rows(sql, ...)` | `List<Row>`, untyped |
| `prepare(sql)` | `Stmt`, outside the cache: `bind(i, v)` (from 1), `step() : bool`, `row()`, `get<T>()`, `exec()`, `query<T>()`, `rows()`, `reset()`, `sql()`; finalized when its last copy goes away |
| `begin()` | `Tx` with `commit()`, `rollback()`, `isDone()`; dropping its last copy without either rolls back, so `?` undoes the work. Inside a transaction `begin` makes a savepoint |
| `Row` | `int` `f64` `str` `blob` `bool` by index (from 0) or name, refusing NULL; `intOrNull` `f64OrNull` `strOrNull` `blobOrNull` `boolOrNull` give `T?`; `value` (a `Value`), `isNull`, `len`, `name`, `index`, `columns`; prints as `{name: 'ryan', points: 1200}` |
| `Value` | `Null`, `Int(int)`, `Real(f64)`, `Text(str)`, `Blob(List<u8>)`; `typeName`, `isNull`, `sql` (an SQL literal) |
| `SqlError` | `code` (SQLite's extended result code, or `UNAVAILABLE`), `message`, `primary()`; constants `OK` ... `NOTADB`, `CONSTRAINT_UNIQUE`, `CONSTRAINT_NOTNULL`, `CONSTRAINT_PRIMARYKEY`, `CONSTRAINT_FOREIGNKEY`, `CONSTRAINT_CHECK` |

**Parameters** bind by type: integers as INTEGER (`u64` as its bits),
`f64`/`f32` as REAL, `bool` as 0 or 1, `str` as TEXT, `List<u8>` as BLOB,
a null `T?` or `Value.Null` as NULL, and a `Value` as what it holds. The
count must match the statement's (`RANGE` otherwise). `exec`, `query` and
`rows` bind without copying (`SQLITE_STATIC`) and reset the statement
before returning; `Stmt.bind` copies.

**Columns into fields**: integer fields take INTEGER, checked to fit the
field's type; `f64` and `f32` take REAL or INTEGER; `bool` takes INTEGER
(nonzero is true); `str` takes TEXT; `List<u8>` takes BLOB or TEXT's bytes;
`T?` of these also takes NULL. A mismatch, or NULL in a field that is not
nullable, is a `SqlError` with code `MISMATCH`; the struct must not have
virtual methods or be packed. The library reads the struct's layout from
`core.fields<T>()` and matches it with the columns once per cached
statement, then fills each row in place with no per-row lookups.

**Threads and `noMutex`.** A `Db` is for one thread (or task) at a time:
its statement cache takes no lock. SQLite still locks the connection's
own mutex on every call unless it is opened with
`Options { noMutex: true }` (`SQLITE_OPEN_NOMUTEX`, multi-thread mode, as
rusqlite opens every connection), which saves that cost (a profile of a
server page running a few small queries put it at a tenth of its CPU). The connection may then
still move between threads, but must never be used by two at once; leave
`noMutex` off when C code or a callback may use it from another thread
at the same time. A SQLite built single-threaded (`SQLITE_THREADSAFE=0`)
has no mutex either way.

```bismuth
let db = sqlite.open("app.db", { noMutex: true })?;
```

**The statement cache** keeps up to `cacheSize` prepared statements per
connection, keyed by SQL text, and evicts the least recently used. Build
SQL with `?` parameters rather than interpolation so one text is reused
(and so values can't change the SQL).

Not supported yet: user-defined SQL functions, the backup and incremental
blob APIs, and `sqlite3_interrupt`.

## h264

`import std.media.h264;` encodes video with OpenH264, Cisco's H.264
encoder, built into the program: using `Encoder` is what makes bic compile
`third_party/openh264` (2.7.0, C++ without its assembly) with the build
machine's C++ compiler, once (cached), and link it statically. No
`libopenh264` and no `libstdc++` are needed where the program runs. Linux
(x86-64 and arm64) and macOS (arm64 and x86-64) only so far: on Windows
`Encoder.new` returns `H264Error` with code `UNAVAILABLE` and `available()`
is false.

```bismuth
import std.media.h264;

let enc = h264.Encoder.new({ width: 1280, height: 720, fps: 30.0, bitrate: 2_000_000, keyInterval: 60 })?;
let f = enc.encode(i420, ms)?;     // Frame: data (Annex B), key, skipped
if f.key { /* starts with SPS and PPS */ }
enc.forceKey();                    // the next frame is an IDR frame
```

| Item | What |
|---|---|
| `Encoder.new(Options)` | `Options { width, height, fps = 30.0, bitrate = 1000000, screen = false, keyInterval = 0 }`: even width and height, bit-rate control, camera or screen content (`screen`), an IDR frame every `keyInterval` frames (0: the first only). Copies share the encoder, destroyed with the last |
| `encode(i420, ms)` | a frame of I420 bytes (Y, then U and V at half size: `width*height*3/2`) with its time stamp; returns `Frame { data, key, skipped }`, `data` the NAL units with start codes, empty when the rate control skips the frame |
| `forceKey()` | make the next frame an IDR frame |
| `H264Error` | `code` (OpenH264's, or `UNAVAILABLE`, `INVALID`), `message` |

The binding calls OpenH264's C API (`WelsCreateSVCEncoder`, then the
`ISVCEncoder` methods through its vtable, as `std.interop.com` does) and
quiets its log. Not yet: the decoder, `SEncParamExt` (spatial layers,
slices, profiles), and Windows and macOS.

