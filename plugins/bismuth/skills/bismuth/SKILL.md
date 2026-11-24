---
name: bismuth
description: The Bismuth programming language (.bi files) and its compiler bic. Use whenever you read, write, review, debug or explain Bismuth code or any .bi file, run bic (build, run, check, fmt, ts, shader, bindgen), or the user mentions Bismuth. Bismuth looks like Rust, C#, Go and Swift but differs in ways that break code written from memory of them (types before names, ref at call sites, T ! E results without Ok/Err, escaped braces in strings, f32 float literals), so load this before writing any.
---

# Bismuth

Bismuth (`.bi`) is a strictly typed, brace-delimited, compiled language for
games, engines, tools and servers. `bic` compiles a program and the
standard library (embedded in bic) straight to a native executable: ELF on
Linux, PE on Windows, Mach-O on macOS, cross-compiling with `--target`.
No garbage collector: values live on the stack or inline, heap objects are
reference counted, and there are arenas for bulk data.

Write Bismuth from this guide, not from Rust or C# habits. When unsure,
write a small file and run `bic check` on it: the compiler's errors are
specific and usually say what to write instead.

## The rules that trip people up

| Wrong (habit from elsewhere) | Right in Bismuth |
|---|---|
| `let x: i32 = 1;` `fn f(x: i32) -> i32` | Types go **before** names: `let i32 x = 1;`, `fn f(i32 x) : i32 { ... }` |
| `fn f() -> T { }` | `:` before the return type; the body is always a block |
| `return Ok(v);` / `Err(e)` | There are no `Ok`/`Err` constructors: return the value (`v`) or the error (`return MyError.Bad;`). `Ok(v)`/`Err(e)` exist only as **match patterns** |
| `xs.push(1);` | A call that mutates its receiver is marked: `ref xs.push(1);` |
| `heal(player)` for a `ref` parameter | The caller writes `ref` too: `heal(ref player, 10);` |
| `"{"` or `"a {} b"` in a string | `{` starts interpolation: write `\{` and `\}` for literal braces, in every string, including JSON text (`"\{\"a\": 1\}"`) |
| `"a" + b` | No string `+`: interpolate, `"a{b}"` |
| `let x = 1.5;` used as f64 | Untyped float literals are `f32` (`float`); write `let f64 x = 1.5;` or `f64(...)` when you need f64. Untyped integers are `int` (= `i64`) |
| `7.0 / 2`, `i32 + i64`, `f32 = int` | No implicit conversions, even between literals: `7.0 / 2.0`, `f32(n)`, `i64(x)`, `int(f)` (truncates) |
| `print(x)`, `print("a", b)` | `print` takes one `str`: `print("{x}")`; `eprint` for stderr |
| `xs.len` on a `List` then `.len()` on an array | `List`/`Map`/`str` have `len()` (and `.len` works on them too); fixed arrays `[3]int` only have `.len`, and no list methods |
| `let p = new P {}; p.hp = 5;` | `let` bindings are fully immutable, handles too: `var p = new P {};` |
| `null` in a plain `T` | Only `T?` can be null; check it (`if x != null`, `x?.f`, `x ?? d`) before use |
| Ignoring a returned `T ! E` | An unused result is an error: `?`, `.or(v)`, `.expect("why")` or `match` |
| `for x in xs { ref xs.push(..) }` | A loop over a collection can't grow or shrink it; collect changes, apply after |
| `match` without all cases | Matches on enums must be exhaustive or end with `_ => ...` |
| `if let Some(x) = ...` | Narrow with `if x != null { ... }` (x is a plain `T` inside) |
| `impl Trait for T`, interfaces | No traits/interfaces: generic bounds (`T: Ord`), single struct inheritance with `virtual fn`, and duck-typed generics |
| `use`, `#include`, `from x import` | `import std.io.fs;` then `fs.readText(...)`; your own `game/world.bi` is `import game.world;` |

## Commands

```sh
bic run main.bi [args...]      # compile and run; arguments after the file go to the program
bic build main.bi              # write ./main (main.exe for --target windows); -o path
bic build --debug main.bi      # -Og -g; --release (= default -O2); -O0..-O3, -Os; --unchecked drops checks
bic build --target windows main.bi   # cross: linux, windows or macos; --arch arm64 (Linux, macOS)
bic check main.bi              # type-check only, fastest feedback; use it constantly
bic ir main.bi                 # print the optimized IR
bic fmt -w .                   # format every .bi file under . (-l lists, --check fails CI)
bic ts -o api.d.ts api.bi      # TypeScript declarations for types marked @ts
bic bindgen lib.h -o lib.bi    # Bismuth declarations for a C header
bic shader -o out game.bi      # compile @vertex/@fragment/@compute stages to SPIR-V, MSL, HLSL
bic lsp                        # language server on stdio (also the bils binary)
bic version                    # this bic and the installed one
```

Flags may go before or after the file, except with `bic run`, where what
follows the file is the program's. `--no-lint` silences naming warnings.
There is no `bic test`: a test is a program whose `main` checks with
`assert(cond, "msg")` (or returns an error), run with `bic run`; a failed
assert panics and the program exits non-zero. One file is the entry point;
it imports the rest.

## Program shape

```bismuth
import std.os;

const GREETING = "hello";             // constants: SCREAMING_CASE

fn shout(str s) : str { s.toUpper() }  // the last expression without ; is the value

fn main() {
    let args = os.args();             // args[0] is the program
    var count = 0;                    // var: reassignable; let: fixed
    for a in args {
        count += 1;
    }
    print("{shout(GREETING)}, {count} args");
}
```

- Every statement ends in `;`. A brace-bodied construct (`fn`, `struct`,
  `if`, `for`, `while`, `match` as a statement) needs none after `}`.
- A block's value is its last expression written without `;`; a block
  ending in `;` is `void`. So `fn f() : int { 1 }`, and `return` is only
  for early exits.
- `fn main()` may return `void ! E`: an error that reaches the top is
  printed and the program exits with a failure status. `os.exit(code)`
  exits at once.
- Naming is linted: types and enum variants PascalCase, functions,
  variables and fields camelCase, constants SCREAMING_CASE, modules
  lowercase.

## Types and literals

| | |
|---|---|
| Integers | `i8 i16 i32 i64 u8 u16 u32 u64`; `int` = `i64`; `byte` = `u8`. Arithmetic wraps; `T.max`, `T.min` |
| Floats | `f32 f64`; `float` = `f32` |
| Other | `bool`, `char` (`'x'`, `'\n'`), `str` (UTF-8, immutable), `void` |
| Math | `vec2 vec3 vec4 ivec2 ivec3 mat3 mat4 quat`, `Angle` (`90.deg`, `1.57.rad`), `Duration` (`2.s`, `16.ms`, `5.us`), `4.kb`/`4.mb`/`1.gb` (int) |
| Literals | `42`, `0xFF`, `0b1010`, `1_000_000`, `3.14`, `1e-6`, `2.0f32`, `r"raw \{no interpolation}"`, ranges `0..10` and `0..=10` |
| Arrays and lists | `[1, 2, 3]` alone is a fixed array `[3]int`; where a `List<T>` is expected (annotated `let`, parameter, field) the same literal builds a `List` |
| Tuples | `(int, str)`, `(1, "one")`, `.0` `.1`, `let (a, b) = t;` |
| Nullable | `T?`; `null` |
| Result | `T ! E` (= `Result<T, E>`) |
| Functions | `fn(int, str) : bool` |

Conversions are explicit and checked: `f32(n)`, `int(x)` (float to int
truncates toward zero, panics if it doesn't fit), `u8(300)` is a compile
error for a constant and a panic at run time; `u8.wrap(300)` is 44,
`u8.clamp(x)`, `u8.check(x)` returns `u8 ! RangeError`. Ranged and
distinct types:

```bismuth
type Hp = int in 0..=100;          // checked wherever a value enters it
distinct type EntityId = u16;      // same layout, can't be mixed with u16

fn main() {
    let Hp full = 100;
    let hp = Hp.clamp(250);        // 100
    let EntityId id = 7;
    let n = u16(id);               // leaving the distinct type is explicit
    print("{full} {hp} {id} {n} {Hp.check(-1).isErr()}");
}
```

## Strings and interpolation

```bismuth
fn main() {
    let name = "ryan";
    let hp = 70;
    let pi = 3.14159;
    print("{name} has {hp} hp");                 // any expression inside { }
    print("{pi:.2} {hp:>5} {255:x} {5:b} {0.256:.1%} {name:?}");
    print("literal braces: \{ and \}");         // \{ \} for { }
    print("json: \{\"name\": \"{name}\"\}");    // so JSON text needs them too
    print("quote \" backslash \\ tab \t end");   // also \n, \r, \0
    let path = r"C:\temp\{raw}";                // raw: no escapes, no interpolation
    let multi = "a string may
span lines";
    print("{path} {multi.lines().len()} {"nested {name}".len()}");
}
```

- Format spec after `:` is `[[fill]align][+][#][0][width][.precision][kind]`:
  `.2`, `>8`, `<8`, `*^9`, `05`, `+`, kinds `x X b o e f g %`, `hex` for
  byte lists, `?` for the debug form (strings quoted).
- Every type prints: structs, enums, lists, maps, tuples, nullables and
  vectors have generated formatting (`"{player}"`).
- `str` methods: `len` `isEmpty` `byteAt` `sub(a, b)` (byte offsets) `find`
  (`int?`) `rfind` `contains` `startsWith` `endsWith` `split(sep)` `lines`
  `trim` `trimStart` `trimEnd` `toUpper` `toLower` `replace(a, b)` `repeat`
  `chars` `charCount` `bytes` `parseInt` (`int ! ParseError`)
  `parseFloat` `cstr`. `==` compares contents; `<` orders.
- Build text with `StrBuf`: `var b = StrBuf.new(); ref b.write("x");
  ref b.writeChar('y'); let s = b.str();` (`ref b.take()` moves it out).
- `char`: `isDigit` `isAlpha` `isAlnum` `isSpace` `isUpper` `toUpper`
  `code`.

## Variables, copies, `ref` and `view`

Every argument is a **copy** unless it says otherwise, and a call that can
change the caller's data says so at the call site.

| Parameter | Meaning | Call |
|---|---|---|
| `T x` | a copy the function owns and may change | `f(x)` |
| `ref T x` | mutable reference to the caller's value | `f(ref x)` |
| `view T x` | read-only reference, no copy (large structs) | `f(x)` |

```bismuth
struct Player { str name; int hp = 50; }

fn heal(ref Player p, int amount) { p.hp += amount; }
fn healed(Player p, int amount) : Player { p.hp += amount; p }   // changes its copy
fn describe(view Player p) : str { "{p.name} ({p.hp} hp)" }

fn main() {
    var player = Player { name: "ryan" };
    let preview = healed(player, 10);
    heal(ref player, 5);
    print("{describe(preview)} {describe(player)}");   // ryan (60 hp) ryan (55 hp)
}
```

- Assigning a struct, `List`, `Map` or `str` copies it (`let q = p;`), so
  pass big ones by `view` (the compiler warns over 64 bytes) and clone
  explicitly where you mean it (`xs.clone()`).
- Two `ref` arguments of one call must not overlap (`f(ref xs, ref xs[0])`
  is an error).
- `ref` method calls: `ref p.takeHit(3);`, `ref m.set(k, v);`. A `?` after
  one applies to its result: `ref f.write(text)?`.

## Structs, methods, enums, match

```bismuth
enum Team { Red, Blue, Spectator }

struct Player {
    str  name;
    Team team = .Spectator;          // defaults; .Variant where the type is known
    int  hp   = 100;
    int? target = null;
    priv int secret = 0;             // priv: hidden outside this module

    fn isAlive(view self) : bool { self.hp > 0 }
    fn takeHit(ref self, int dmg) { self.hp = max(self.hp - dmg, 0); }
}

impl Player {                        // methods can also go in impl blocks
    fn named(str name) : Player { Player { name: name } }   // no self: static
}

enum Shape {
    Circle(f32 r),
    Rect(f32 w, f32 h),
}

fn area(Shape s) : f32 {
    match s {
        Circle(r) => PI * r * r,
        Rect(w, h) => w * h,
    }
}

fn main() {
    var p = Player.named("ann");
    ref p.takeHit(30);
    let List<Shape> shapes = [Shape.Circle(1.0), Shape.Rect(2.0, 3.0)];
    for s in shapes { print("{s}: {area(s):.2}"); }
    let label = match p.hp {
        0 => "dead",
        1..=25 => "critical",
        _ => "ok",
    };
    print("{p} {label} {p == Player.named("ann")}");
}
```

- Struct literals name fields: `Player { name: "x", hp: 3 }`; fields with
  defaults may be left out. `==`, hashing, printing and copying are
  generated for every struct, so structs work as `Map` keys.
- Receivers are explicit: `self` (copy), `view self`, `ref self`.
- `match` is an expression; arms are separated by `,`; an arm body can be
  a block (`Err(e) => { log(e); 0 },`). Patterns: literals, ranges,
  strings, `null`, `_`, enum variants binding their fields, `Ok(v)`/`Err(e)`
  on results, and constants. No nested destructuring.
- Operators are methods with known names: `opAdd opSub opMul opDiv opMod
  opNeg`, `opCmp(self, T o) : int` for `< <= > >=`, `format(view self, ref
  StrBuf b, str spec)` for printing, `next(ref self) : T?` to be iterated
  with `for`, `drop(ref self)` runs at end of scope.
- Inheritance: `struct Spinner : Component { ... }` extends one struct.
  `virtual fn update(ref self, f32 dt) { }` may be overridden by declaring
  the same method; `virtual fn name(view self) : str;` must be. Families
  live in `List<Shared<Component>>`; `x is Spinner`, `x as Spinner`
  (`Shared<Spinner>?`), `super.update(dt)`.

## Control flow

```bismuth
fn main() {
    for i in 0..3 { print("i = {i}"); }               // 0..n exclusive, 0..=n inclusive
    let List<str> names = ["ann", "bob"];
    for i, name in names { print("{i}: {name}"); }   // index and element
    var List<int> xs = [1, 2, 3];
    for x in ref xs { x *= 10; }                      // by reference: change elements
    var m = Map<str, int>();
    ref m.set("a", 1);
    for k, v in m { print("{k}={v}"); }
    var n = 0;
    while n < 10 {
        n += 3;
        if n == 6 { continue; }
        if n > 7 { break; }
    }
    loop { n -= 1; if n < 0 { break; } }              // loop until break
    let kind = n % 2 == 0 ? "even" : "odd";           // ternary
    let size = if xs.len() > 2 { "big" } else { "small" };   // if is an expression
    print("{xs} {kind} {size}");
}
```

`defer stmt;` runs when the scope exits, on every path (`return`, `?`,
`break`).

## Errors and nulls

```bismuth
enum ParseErr { Empty, NotANumber(str text) }
enum AppError { from ParseErr, TooBig }      // `from`: ? converts ParseErr into AppError

fn parseCount(str text) : int ! ParseErr {
    let t = text.trim();
    if t == "" { return ParseErr.Empty; }    // return the error value itself
    match t.parseInt() {
        Ok(n) => n,                          // and the plain value for success
        Err(e) => ParseErr.NotANumber(t),
    }
}

fn total(str a, str b) : int ! AppError {
    let x = parseCount(a)?;                  // ? returns the error to the caller
    let y = parseCount(b)?;
    if x + y > 1000 { return AppError.TooBig; }
    x + y
}

struct User { str name; str? nickname = null; }

fn main() {
    print("{total("40", "2").or(0)} {total("x", "1").isErr()}");
    match total("", "1") {
        Ok(n) => print("total {n}"),
        Err(e) => print("error {e}"),        // error ParseErr.Empty
    }
    let List<User> users = [User { name: "ann", nickname: "a" }, User { name: "bob" }];
    let bob = users.find(u => u.name == "bob");   // User?
    print(bob?.nickname ?? "no nickname");        // ?. short-circuits, ?? defaults
    if bob != null {
        print("found {bob.name}");                // narrowed: bob is a User here
    }
}
```

- Results: `f()?`, `.or(fallback)`, `.orElse(e => ...)`, `.expect("msg")`
  (panics), `.isOk()`, `.isErr()`, `.ok()` (`T?`), `.err()`, `match`.
- The error type can be any type: an enum (usual), or `str` for quick
  code (`fn f() : int ! str { return "bad input"; }`). When the value and
  the error are the same type (`str ! str`), a bare value is taken as the
  value and bic warns: write `.Ok(v)` or `.Err(e)`.
- `?` only converts between error types that declare it with `from`.
- `x!` unwraps a nullable or crashes (the linter flags it). Narrowing works
  for locals and copy parameters in `if x != null`, `while x != null`, the
  else of `if x == null`, and after `if x == null { return; }`.

## Generics and lambdas

```bismuth
fn largest<T: Ord>(T a, T b) : T { a > b ? a : b }

struct Stack<T> {
    List<T> items;
}

impl<T> Stack<T> {
    fn new() : Stack<T> { Stack<T> { items: List<T>() } }
    fn push(ref self, T item) { ref self.items.push(item); }
    fn pop(ref self) : T? { ref self.items.pop() }
}

fn apply(fn(int) : int f, int x) : int { f(x) }

fn main() {
    var s = Stack<str>.new();
    ref s.push("a");
    let top = ref s.pop();
    let k = 10;
    let add = (int a, int b) => { let sum = a + b; sum * 2 };   // block body
    print("{largest(3, 9)} {largest("fig", "pear")} {top ?? "-"} {apply(n => n + k, 1)} {add(1, 2)}");
}
```

- Bounds: `Ord`, `Eq`, `Hash`, `Num`, `Int`, `Float`, `Copy`, or a struct
  name (that struct or one extending it). Without a bound, a generic body
  is checked per instance, so `T x` can use whatever the type has.
- Generics are monomorphized; `f<int>(...)` gives type arguments
  explicitly, leading ones may be given and the rest inferred.
- Lambdas: `x => expr`, `(a, b) => expr`, `(int a) => { ...; value }`.
  They capture locals by reference, so a lambda must not outlive what it
  captures (returning or storing one that does is an error).
- Subscribable actions: `std.actions` has `Action<T>`, usually a field
  (`actions.Action<int> scored = {};`: fields need their default written)
  with `ref s.scored.subscribe(fn(view T)) : u64`, `ref s.scored.unsubscribe(id)`
  and `s.scored.emit(value)`. A listener outlives the subscribing call, so
  it can't capture locals or a method's `self`: keep shared state in a
  `Shared`.

```bismuth
import std.actions;

struct Player {
    int points = 0;
    actions.Action<int> scored = {};

    fn score(ref self, int n) {
        self.points += n;
        self.scored.emit(n);
    }
}

struct Hud { int shown = 0; }

fn main() {
    var p = Player {};
    var hud = new Hud {};
    let id = ref p.scored.subscribe(n => { hud.shown += n; });
    ref p.score(10);
    ref p.scored.unsubscribe(id);
    ref p.score(5);
    print("{p.points} {hud.shown}");   // 15 10
}
```

## Collections

```bismuth
import std.collections;

fn main() {
    var List<int> xs = [3, 1, 2];
    ref xs.push(4);
    ref xs.sort();
    let evens = xs.filter(x => x % 2 == 0);           // a new List
    let total = xs.map(x => x * 10).sum();
    let first = xs.find(x => x > 1);                  // int?
    let desc = xs.sortBy((a, b) => a > b);            // a sorted copy; the lambda is "a before b"
    print("{xs} {evens} {total} {first ?? -1} {desc} {xs.contains(2)} {xs[0]} {xs.len()}");
    let last = ref xs.pop();                          // int?, removes it

    var ages = Map<str, int>();
    ref ages.set("ann", 31);
    let a = ages.get("ann") ?? 0;                     // get returns V?
    print("{a} {ages.has("bob")} {ages.keys()} {last ?? 0}");

    var seen = collections.Set<str>();
    ref seen.add("x");
    print("{seen.has("x")} {xs.asSlice().sub(1, 3).toList()}");

    // a Slice only reads; writing the elements takes a MutSlice
    xs.asMutSlice().sortInPlace((a, b) => a > b);
    zero(xs.asMutSlice().sub(0, 1));
    print("{xs} {sumOf(xs.asMutSlice())}");       // a MutSlice passes as a Slice
}

fn zero(MutSlice<int> s) { for x in ref s { x = 0; } }
fn sumOf(Slice<int> s) : int { s.sum() }
```

- `List<T>`: `push` `pop` `get` (`T?`) `set` `insert` `remove` `swapRemove`
  `clear` `truncate` `append` `drain(a, b)` (removes [a, b), returns
  them) `reverse` `sort` `sortByInPlace` `stableSort`
  `stableSortByInPlace` `first` `last` `clone` `asSlice` `asMutSlice`,
  plus every `Slice` method:
  `filter` `map` `each` `fold(init, f)` `find` `findIndex` `any` `all`
  `contains` `indexOf` `count` `countWhere` `sum` `min` `max` `sortBy`
  `stableSortBy` `sorted` `reversed` `join(sep)` `sub(a, b)` (a `Slice` view) `toList`.
  `xs[i]` is bounds-checked (panics when out of range).
- `Slice<T>` (`xs.asSlice()`, `xs.sub(a, b)`) is a read-only view:
  `s[i] = v` and `for x in ref s` through it are errors, wherever it came
  from. `MutSlice<T>` (`xs.asMutSlice()`) is the writable one: element
  writes, `for x in ref s`, `reverseInPlace`, `sortInPlace`, a `sub` that
  stays a `MutSlice`, and every `Slice` method. `asMutSlice()` needs a list
  that may change (`var`, `ref` or by-value parameter), not a view
  parameter or let binding. A `MutSlice<T>` passes as a `Slice<T>`
  implicitly, never the reverse: take `Slice<T>` to read, `MutSlice<T>` to
  write. In shaders they are read-only and read-write storage buffers.
- `Map<K, V>`: `set` `get` (`V?`) `has` `remove` `keys` `values` `len`
  `clear`; `for k, v in m`.
- `std.collections`: `Set` `Deque` `Queue` `Stack` `PriorityQueue`
  `LinkedList` `SortedMap` `SortedSet` `BitSet` `Counter` `MultiMap`.
  `std.linq` adds lazy queries: `xs.query().where(...).select(...).toList()`.

## Memory

| Kind | Lives | Freed when |
|---|---|---|
| primitives, structs, vectors | stack or inline in their container | scope ends |
| `List`, `Map`, `str`, `StrBuf` contents | a heap buffer the value owns | the owner is dropped |
| `new T { ... }` | heap, reference counted: `Shared<T>` | the last handle drops |
| `arena.alloc<T>(n)` | an `Arena` you own (plain data only) | `arena.reset()` / `arena.free()` |
| `mem.alloc<T>(n)` | raw heap, inside `unsafe` | `mem.free(p)` |

```bismuth
struct Node {
    str name;
    List<Shared<Node>> children;
    Weak<Node>? parent = null;            // Weak breaks reference cycles
}

struct Particle { vec3 pos; f32 life; }

fn main() {
    let root = new Node { name: "root", children: List<Shared<Node>>() };
    var leaf = new Node { name: "leaf", children: List<Shared<Node>>(), parent: root.weak() };
    leaf.name = "leaf2";                  // writes through the handle need a var
    print("{leaf.name}'s parent: {leaf.parent?.get()?.name ?? "none"}");

    let frame = Arena(1.mb);
    defer frame.free();                   // runs on every exit path
    var ps = frame.alloc<Particle>(256);  // ArenaSlice<Particle>
    for p in ref ps { p.life = 1.0; }
    print("{ps.len()} {ps.asSlice().countWhere(p => p.life > 0.5)}");
    frame.reset();                        // using ps after this panics
}
```

Copies of a `Shared<T>` share one object: `a == b` and `a.ptrEq(b)` are
true for handles to the same object, and a `Shared<T>` passes where a
`view T` is expected. Raw pointers `*T`, pointer
arithmetic, `mem.alloc` and unsafe C calls go inside `unsafe { }`;
`fn f() : T unsafe { ... }` makes a body unsafe but stays safe to call,
`unsafe fn f()` makes callers need `unsafe`.

## Concurrency

```bismuth
async fn slowSquare(int x) : int ! str {
    sleep(20.ms);
    if x < 0 { return "negative input"; }
    x * x
}

fn crunch(int n) : int {
    var s = 0;
    for i in 0..n { s += i % 7; }
    s
}

fn produce(Chan<str> ch, int n) {
    for i in 1..=n { ch.send("job {i}"); }
    ch.close();
}

fn main() : void ! str {
    let t = spawn crunch(1000);                             // any function; returns Task<int>
    let a = await slowSquare(4)?;                           // call an async fn and wait
    let (b, c) = await all(slowSquare(5), slowSquare(6))?;  // both, or the first error
    let winner = await race(timeout(1.ms), spawn slowSquare(9));   // index of the first done
    print("{a} {b} {c} {await t} {winner}");

    let ch = Chan<str>(4);                                  // bounded channel
    let producer = spawn produce(ch, 2);
    for job in ch { print(job); }                           // ends when closed and drained
    await producer;

    let hits = Atomic<int>(0);
    let log = Mutex<List<int>>(List<int>());
    parallel for i in 0..100 {                              // iterations run on all cores
        hits.add(1);
        if i % 50 == 0 { log.lock(l => { ref l.push(i); }); }
    }
    var List<f32> vals = [1.0, 2.0, 3.0];
    parallel for v in ref vals { v *= 2.0; }                // each iteration writes its own element
    parallel for v in vals reduce (+) big = 0 {             // per-job copies, merged
        if v > 2.0 { big += 1; }
    }
    print("{hits.load()} {log.get().len()} {big}");
}
```

- `spawn f(args)` starts a task; a task never awaited is cancelled when its
  scope exits. `task.done()`, `task.cancel()`; `join(a, b)` is `all` for
  tasks whose values aren't results; `select(tasks)` gives the index of the
  first done; `blocking(() => ...)` around blocking C
  calls; `yieldNow()`.
- An async method that changes its receiver: `await ref conn.read()`.
  `for await msg in server.messages<T>()` consumes a stream.
- The compiler rejects writes to shared data from a `parallel for` body or
  a spawned task (including through a `Shared` handle): use `reduce`,
  `Atomic<T>` (`load` `store` `add` `sub` `swap` `compareSwap`),
  `Mutex<T>` (`lock(fn(ref T))` `get` `set`) or `Chan<T>` (`send` `recv`
  `tryRecv` `close`, `sender()`). Copies of these share one object, so pass
  them to tasks by value. `std.sync` has `Oneshot` `Notify` `Semaphore`
  `RwLock` `Watch` `Broadcast` `CancelToken` `WaitGroup` `Barrier`.

## Modules and the standard library

A file is a module; a folder path is a dotted name. `import game.world;`
reads `game/world.bi` beside the main file and is used as `world.f()`;
`import std.collections as c;` renames. Top-level names are public unless
`priv`. When there is no `world.bi`, `world_linux.bi` / `world_windows.bi` /
`world_darwin.bi` (or `world_unix.bi`) is taken for the target. Names
under `std` are always the standard library.

The program root is the folder of the file you build or run: every import
resolves from there (then from each `-I dir`), never from the importing
file's own folder. So a project's programs (the app,
its test runner, a build script, benches) all sit in that root folder,
and the modules they share sit in folders under it. By convention the
entry is `main.bi` (or the program's name), a module's tests are
`name_test.bi` beside `name.bi` with the data they read in `testdata/`
beside them, and one test program at the root imports and runs them,
since there is no `bic test`.

A project of several executables and libraries can be built with
**bimake** (`bic make`): a `BiMakeLists.txt` at its root, in CMake's
language (`project(app)`, `add_executable(app main.bi)`,
`add_library(engine STATIC engine/api.bi)`, `target_link_libraries(app
PRIVATE engine)`), then `bimake -S . -B build && bimake --build build`.
A linked library's folder becomes an import folder (`-I`); the first
source of a target is the file bic builds. `docs/bimake.md` has the
commands.

The prelude (`std.core`) needs no import: `print` `eprint` `printRaw`,
`panic` `assert` `unreachable` `todo`, `List` `Slice` `Map` `str`
`StrBuf`, `Shared` `Weak` `Arena`, `Result`, number methods (`abs` `min`
`max` `clampTo` `pow` `sqrt` `floor` `round` `trunc`, generic `min` `max` `clamp`
`lerp` `swap`, `PI` `TAU` `E`), vectors and matrices (`dot` `cross`
`length` `normalized` `distance`, swizzles `v.xz`), `Angle` and trig,
`Duration`, `Task` `spawn` `await` `all` `join` `race` `timeout` `sleep`
`Atomic` `Mutex` `Chan`.

| Import | For |
|---|---|
| `std.os` | `args()`, `env(name) : str?`, `setEnv`, `exit(code)`, `cpuCount()` |
| `std.os.process` | `run(["git", "status"]) : Output ! ProcessError` (`status.code`, `stdout`, `stderr`), `start` for a `Child` |
| `std.os.signal`, `std.os.unix`, `std.os.windows`, `std.os.windows.d3d11`/`.dxgi`, `std.os.dbus` | signals, ioctl/TUN, registry/services, Direct3D 11 and DXGI, D-Bus |
| `std.io.fs` | `readText(path) : str ! IoError`, `read` (bytes), `write(path, text)`, `writeBytes`, `append`, `exists`, `listDir`, `makeDirAll`, `remove`, `removeAll`, `rename`, `stat`, `open` |
| `std.io.stdin` | `readLine() : str?`, `lines()`, `readAll()`, `prompt(q)` |
| `std.time` | `now()` (`.elapsed()`), `unixMillis()`, `Date` `DateTime` `Zone`, RFC 3339, `strftime` |
| `std.cli` | flag parsing: `Command.new(name, about)`, `boolFlag` `strFlag` `intFlag` `arg`, `parseOs()` |
| `std.encoding.json` | `json.encode(value)`, `json.decode<T>(text) : T ! JsonError`, `parse(text) : Value` (`v["k"][0].int() ?? 0`) |
| `std.encoding.*` | `csv` `toml` `yaml` `hex` `base64` `pem` `der` |
| `std.net` | messages (`listen`, `connect`, `send`, `messages<T>()`), TCP/UDP streams; `std.net.http` (client `http.get(url)` and `Server`), `std.net.tls`, `std.net.ws`, `std.net.url`, `std.net.ssh`, `std.net.quic`, `std.net.http3` |
| `std.db.sqlite` | `open(path) : Db ! SqlError`, `db.exec(sql, args...)`, `db.query<T>(sql, args...) : List<T>` (rows into a struct by column name) |
| `std.crypto`, `std.crypto.x509`, `std.uuid`, `std.rand` | hashes, HMAC, AEADs, keys, certificates, UUIDs, random numbers |
| `std.collections`, `std.linq`, `std.text.regex` | more collections, lazy queries, regular expressions |
| `std.sync`, `std.stream` | more task coordination, stream utilities |
| `std.actions` | subscribable actions: an `Action<T>` field with `subscribe` `unsubscribe` `emit` |
| `std.io.compression.*` | `deflate` `zlib` `gzip` `lz4` |
| `std.window`, `std.gfx`, `std.gfx.image`, `std.gfx.model`, `std.ui`, `std.audio`, `std.gamepad`, `std.noise`, `std.collision` | windows and input, GPU rendering (Vulkan, D3D11/12, Metal), images and PNG, 3D models, immediate-mode GUI, audio, gamepads, noise, collision |
| `std.interop.dl`, `std.interop.com`, `std.interop.winrt`, `std.interop.objc` | C libraries at run time, COM (calling and implementing), WinRT, Objective-C |

## C interop, layout, shaders

- `@link("m") extern c fn cos(f64 x) : f64;` declares a C function (no
  `@link`: the C library). `bic bindgen lib.h -o lib.bi` writes the
  declarations; check them in. `str` arguments go to C as `const char*`.
  `export c fn tick(f32 dt) { }` is callable from C. Functions taking raw
  pointers are `unsafe extern c fn`, called inside `unsafe`.
- `struct S packed { ... }`, `packed(be)` / `packed(le)` for wire formats;
  `S.fromBytes(bytes)` and `s.bytes()`; `comptime assert sizeof(S) == 14;`;
  `@align(16)` and `@pad(4)` per field.
- `@vertex`, `@fragment` and `@compute(x, y, z)` functions are GPU shader
  stages in a subset of the language; `core.shader(f)` gives their
  compiled code.
- `core.includeStr("file.txt")` is a file's text, read at compile time
  (path relative to the source file); it can initialise a `const`.
- Attributes: `@ts` (TypeScript declarations), `@serde(...)` (JSON/YAML
  field options), `@allocator` (a program's own heap). `@ts` gives a type
  with its own `serialize<W>` as `unknown`: put `@ts(as: "...")` on the
  type to give its shape (braces in it escaped, `\{ ... \}`).
- In the spec but not implemented yet: hot reload (`@hot` compiles as a
  plain function, with a warning), `net`'s `codec: .json` and `tickRate`,
  HTTP/2, native arm64 code for Windows. `docs/compiler.md` lists what is
  missing.

## Pitfalls checklist

Before finishing Bismuth code, check:

1. Types before names everywhere; `:` before return types.
2. `ref` on mutating method calls and on `ref` arguments.
3. Every `{` or `}` meant literally inside a string is `\{` / `\}`.
4. No `Ok(...)` / `Err(...)` calls (a value or an error converts; `.Ok(v)` /
   `.Err(e)` only where both sides have its type); no `+` on strings.
5. Float literals are `f32` unless typed; no int/float mixing; explicit
   conversions between every numeric type.
6. Results are handled; nullables are checked before use.
7. `var` for anything assigned or mutated after creation, handles included.
8. Collections aren't changed while a `for` loop walks them.
9. Each `std` module used is imported (`import std.io.fs;`); prelude items
   aren't.
10. `bic check file.bi` passes (and `bic fmt -w` for layout).

## More

The full documentation is in `docs/` beside this file when the skill was
installed by `bic install`, or `docs/` at the root of the Bismuth
repository (https://git.philab.io/ryan/bismuth):

- `docs/spec.md`: the language definition
- `docs/stdlib.md`: every std module, type and function
- `docs/tooling.md`: bic's commands, install/upgrade, fmt, ts, the language server
- `docs/c-interop.md`, `docs/shaders.md`, `docs/graphics.md`, `docs/compiler.md`

Read the relevant one before using an API this guide doesn't show; don't
guess std signatures. The repository's `samples/` and `wiki/` have
complete, runnable programs.
