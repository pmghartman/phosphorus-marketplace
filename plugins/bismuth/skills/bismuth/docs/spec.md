# Bismuth Language Spec

*Nov 18, 2026 · @Ryan — design draft*

## Overview

Bismuth (`.bi`) is a strictly typed, brace-delimited language for games, engines, tools and infrastructure. Its syntax comes from a preference survey.

Design principles:

- **No hidden magic.** There are no implicit conversions or hidden allocations. Mutation is visible at the call site.
- **Strict types, little ceremony.** The compiler checks a lot while you write few type annotations.
- **Layout you control.** Structs can be packed with no padding, and you never fight the compiler for it.
- **Concurrency for games.** `async`/`await` for I/O and `parallel for` for data work are both built in.
- **Common things are short.** Printing, networking and C interop each take one line in the common case.
- **Pragmatic.** Plain functions, structs with methods and functional-style chains all fit; no paradigm is forced.

```bismuth
fn getValue(float f) : float { return f * 2.0; }
```

## Lexical basics

Every statement ends in `;`. Newlines and indentation never change meaning, so any statement can be written on one line or split across many.

```bismuth
let total = items.filter(x => x.active).map(x => x.cost).sum();

let total = items
    .filter(x => x.active)
    .map(x => x.cost)
    .sum();
```

Those two statements are identical to the compiler.

Semicolon rules:

- Statements, declarations, struct fields and expression-bodied functions end in `;`.
- A brace-bodied construct needs no `;` after its closing `}`. That covers `fn ... { }`, `struct`, `impl`, `if`, `for`, `while` and `match` used as a statement.
- A block's value is its last expression, written without a trailing `;`. A block whose last line ends in `;` has type `void`.
- Match arms and enum variants are separated by `,`, not `;`.

### Comments

```bismuth
// line comment
/* block comment, /* nestable */ */
```

### Naming

The compiler warns when a name breaks the convention for its kind.

| Kind | Convention | Example |
|---|---|---|
| Types, enums, enum variants | PascalCase | `PlayerState`, `Team.Red` |
| Functions, methods, variables, fields | camelCase | `getValue`, `playerSpeed` |
| Constants | SCREAMING_CASE | `MAX_PARTICLES` |
| Modules and namespaces | lowercase, dotted | `std.net.http`, `game.world` |

### Literals

| Literal | Example |
|---|---|
| Integers | `42`, `0xFF`, `0b1010`, `1_000_000` |
| Floats | `3.14`, `1e-6`, `2.0f32` |
| Strings, with interpolation | `"hp {player.hp}"`; `\{` and `\}` are literal braces (`"\{\"hp\": {hp}\}"`; `{{` and `}}` are accepted too) |
| Raw strings | `r"C:\temp\{no interpolation}"` |
| Ranges | `0..10` (exclusive), `0..=10` (inclusive) |
| Vectors | `vec3(1, 2, 3)` |
| Struct literals | `{ proto: .udp, tickRate: 60 }` |

## Types

The type system is strict: no implicit conversions, exhaustive checks, and types that can encode invariants. Local inference keeps annotations rare.

### Primitives

| Category | Types |
|---|---|
| Integers | `i8 i16 i32 i64 u8 u16 u32 u64`; `int` = `i64` |
| Floats | `f32 f64`; `float` = `f32` |
| Other | `bool`, `char`, `str`, `void`, `byte` = `u8` |
| Math | `vec2 vec3 vec4`, `ivec2 ivec3`, `mat3 mat4`, `quat` |

### No implicit conversions

```bismuth
let i32 a = 10;
let f32 b = a;          // error: i32 is not f32
let c = f32(a);         // ok, explicit
let d = u8(300);        // error at compile time: 300 does not fit in u8
let e = u8.wrap(300);   // ok, explicit wrapping
```

A float converts to an integer type by truncating toward zero, and panics when the truncated value does not fit (or is NaN): `u8(255.9)` is 255, `u8(256.0)` panics. `T.clamp(x)` and `T.check(x)` treat a float the same way, and NaN as below the range. Shift counts are taken modulo the operand's width, in constants as at run time, and constants of type `u64` above `int`'s range divide, compare and shift right as unsigned.

### Ranged and distinct types

A ranged type is checked wherever a value enters it. A distinct type has the same layout as its base but cannot be mixed with it.

```bismuth
type Hp = int in 0..=100;
distinct type EntityId = u16;

let Hp hp = 120;              // error: 120 is outside 0..=100
let hp2 = Hp.clamp(raw);      // clamps into range
let hp3 = Hp.check(raw)?;     // returns an error if out of range
let EntityId id = 7;          // ok, literal
let u16 n = id;               // error: EntityId is not u16
```

Arithmetic on a ranged value gives its base type (`hp - dmg` is an `int`). A compound assignment into a ranged variable (`hp -= dmg`) checks the result as it goes in and panics when it is out of range, as `Hp(x)` does; `List<Hp>.sum()` works the same way. A range such as `0..=4` is not a value: it is written where it is used, in a `for` loop, a match pattern or a slice.

### Nullable types

`T?` is either a `T` or `null`. A plain `T` can never be null. See [Errors and nulls](#errors-and-nulls).

### Generics

Generics use angle brackets. Constraints follow `:`; a struct name as a constraint accepts that struct and the structs extending it ([Inheritance](#inheritance-and-virtual-methods)).

```bismuth
fn max<T: Ord>(T a, T b) : T { a > b ? a : b }

struct Pool<T> {
    List<T> items;
    int     used;
}

let ids = List<EntityId>();
```

A call can give the leading type arguments and leave the rest to be inferred from its arguments: given `fn query<T, A>(str sql, A a) : List<T>`, `query<Score>("... where points > ?", 1000)` takes `A` to be `int`.

Each set of type arguments gets its own instance, made at compile time. A generic that uses itself with an ever deeper type argument (`f<List<T>>` inside `f<T>`, or a field `List<A<List<T>>>` in `struct A<T>`) would need endless instances, so type arguments may nest at most 64 levels deep.

## Variables and passing

`let` bindings cannot be reassigned. `var` bindings can. A type is optional and comes before the name, as in parameters.

```bismuth
let name = "ryan";           // inferred str, fixed
var f32 speed = 3.0;         // explicit type, reassignable
speed += 0.5;
name = "other";              // error: name is a let binding
```

### Copy or reference

Every argument is passed by copy unless you write `ref`. A copy belongs to the function, which may change it freely; the caller never sees the change. A `ref` parameter is a mutable reference to the caller's value.

The call site must also write `ref`, so a call that mutates your data is visible where it happens.

The places one call takes by `ref` (the receiver of a `ref` method among them) must not overlap: passing a variable and a field or element of it (`g(ref xs[0], ref xs)`, `h(ref s, ref s.f)`), or one place twice, is an error, since through one the callee could free or move the storage the other points into. Different variables, different fields (`swap(ref p.x, ref p.y)`) and elements at different integer literal indexes are fine; other indexes count as possibly the same element. A place passed to a `view` parameter that lies inside a place the same call takes by `ref` (`f(xs[0], ref xs)`, `ref s.wipe(s.l[0])`) is an error when the callee may change, through that `ref`, the viewed place or what holds it: the view would read freed or moved storage. What a function may change through each `ref` parameter is worked out from its body and the functions it passes the parameter on to; calls into `std`, `extern` functions, virtual methods and function values are taken to change all of what they are given. A field or element of a variable passed by `ref` or `view` together with a lambda that changes the variable, or to a function value holding such a lambda (`g(xs[0])` where `g` clears `xs`), is an error too.

```bismuth
fn heal(ref Player p, int amount) {
    p.hp = Hp.clamp(p.hp + amount);
}

fn healed(Player p, int amount) : Player {
    p.hp = Hp.clamp(p.hp + amount);   // changes the copy only
    p
}

heal(ref player, 10);                 // player changed
let preview = healed(player, 10);     // player unchanged
```

| Form | Meaning | Call site |
|---|---|---|
| `T x` | Copy, owned by the function | `f(x)` |
| `ref T x` | Mutable reference | `f(ref x)` |
| `view T x` | Read-only reference, no copy (for large structs) | `f(x)` |

Copies are explicit in cost: the compiler warns when a copy parameter is larger than 64 bytes and suggests `view`.

A `Shared<T>` handle passes as a `view T` (or as a `view` of a struct `T` extends): the callee reads the object the handle holds, with no copy, so `describe(player)` works whether `player` is a `Player` or a `Shared<Player>`. A `ref T` or copy `T` parameter still needs a `T` place or value.

## Functions

A function is `fn name(params) : ReturnType { body }`. Parameters put the type first, `:` starts the return type, and the body is a block in braces.

```bismuth
fn getValue(float f) : float {
    return f * 2.0;
}
```

### Body forms

The body is always a block. Its value is its last expression, written without a `;`, so a one-line function needs no `return`:

```bismuth
fn lerp(f32 a, f32 b, f32 t) : f32 { a + (b - a) * t }

fn lerp(f32 a, f32 b, f32 t) : f32 {
    let span = b - a;
    a + span * t
}
```

`unsafe` before the brace makes the whole body an `unsafe` block. The function is still safe to call; only its body may use raw pointers. (An `unsafe fn` is the other way round: its callers need `unsafe`.)

```bismuth
fn firstByte(*u8 p) : u8 unsafe { p[0] }
```

Before November 2026, function bodies followed `->`: `fn f() : T -> { ... }`, `fn f() : T -> expr;` and `fn f() : T -> unsafe { ... }`. That form is now an error, and `bic fmt --fix -w .` rewrites it.

With no return type, the function returns `void`:

```bismuth
fn reset(ref Player p) { p.hp = 100; }
```

Such a function's `return` takes no value: `return 5;` in it is an error. `return f();` is allowed when `f` itself returns `void`.

### Default and named arguments

```bismuth
fn spawn(vec3 at, int count = 1, f32 spread = 0.0) { /* ... */ }

spawn(origin);
spawn(origin, count: 40, spread: 30.0);
```

### Lambdas

Lambdas use `=>`. A single-expression lambda needs no braces.

```bismuth
let alive = particles.filter(p => p.life > 0).count();
let byHp  = players.sortBy((a, b) => a.hp < b.hp);
let log   = (str msg) => { print("[game] {msg}"); };
```

A lambda captures the variables it uses by reference, so it must not outlive them: returning a lambda that captures a local or parameter of the function returning it (directly, through a local holding it, inside a struct, list or tuple literal, as an argument of a call whose result type can hold it, such as an enum variant or a helper that builds a struct, or as an argument or the receiver of a spawned call, whose task may run after the function returns) is an error, and so is detaching such a task (`(spawn srv.serve(l)).detach()` where a route of `srv` captures a local): it runs on after the function returns. Passing such a lambda to a parameter the callee keeps in a detached task (it detaches a task holding the parameter, or passes it on to another such parameter, as `std.stream.buffered` does with `f`) is an error at the call for the same reason. So is storing a capturing lambda, by assignment or as an argument of a `ref` method call (`ref fs.push(() => i)`), into a variable declared outside the block that declares a variable it captures, when that variable's type can hold the lambda: a lambda pushed into a list from a loop body would otherwise read the loop's variable after its iteration. Pass such values to the lambda as parameters, or keep them in a `Shared`.

### Methods

Methods are written in the struct's body, after or among its fields, or in an `impl` block of the type (which is how methods are added to enums, to types from other modules, and to every instance of a generic type at once). The receiver is written explicitly as `self`, `ref self` or `view self`, using the same copy and reference rules as any parameter. A function without a receiver is a static function, called on the type (`Counter.make(5)`). A `priv fn` method or static function can only be called from the module that declares it, like a `priv` field.

```bismuth
struct Player {
    str name;
    Hp  hp = 100;

    fn isAlive(view self) : bool { self.hp > 0 }

    fn takeHit(ref self, int dmg) {
        self.hp = Hp.clamp(self.hp - dmg);
    }
}

impl Player {
    fn named(str name) : Player { Player { name: name } }
}

ref player.takeHit(25);     // `ref` marks the mutating call
```

A `?` or `!` after a `ref`-marked call applies to the call's result: `ref f.write(text)?` is `(ref f.write(text))?`.

## Structs, enums and layout

### Structs

Fields are type-first and end in `;`. A field can have a default value.

```bismuth
struct Player {
    EntityId  id;
    str       name;
    vec3      pos;
    Team      team;
    Hp        hp     = 100;
    EntityId? target = null;
}

let p = Player { id: 1, name: "ryan", pos: vec3(0), team: Team.Red };
```

Structs carry no boilerplate. Equality, hashing, printing and copying are generated, and fields are accessed directly with no getters or setters. Use `priv` to hide a field outside its module.

A struct literal can stand in the head of `if`, `while`, `match` and `for ... in` (`for i in Countdown { n: 3 } { ... }`): no block starts with `name:`, so `T { name: ...` there is a literal. An empty `T {}` in a head is read as `T` followed by the body.

### Packed layout

A plain struct uses natural alignment, and its layout is still deterministic and documented. Add `packed` to remove all padding. `packed(be)` or `packed(le)` also fixes the byte order, which is what network and file formats need.

```bismuth
struct Pixel packed {
    u8 r; u8 g; u8 b; u8 a;
}

struct PlayerState packed(be) {
    EntityId id;      // 2 bytes
    vec3     pos;     // 12 bytes
}

comptime assert sizeof(PlayerState) == 14;
comptime assert offsetof(PlayerState, pos) == 2;
```

Per-field control is available when a format needs it:

```bismuth
struct GpuLight {
    vec3 pos;
    @align(16) vec4 color;
    @pad(4);
}
```

A packed struct can be read straight from bytes with `PlayerState.fromBytes(buf)?` and written with `state.bytes()`. Neither call allocates. `fromBytes` returns `DecodeError.TooShort` for too few bytes and `DecodeError.BadData` when a field's bytes are not a value of its type: a `bool` other than 0 or 1, an enum tag no variant has, or a ranged value outside its range.

### Enums

Enums can be plain or carry data.

```bismuth
enum Team { Red, Blue, Spectator }

enum Shape {
    Circle(f32 r),
    Rect(f32 w, f32 h),
}
```

### Loops over iterators

`for x in it` works on any value with `next(ref self) : T?` (or an `iter()` that returns one) and runs until `next` returns null. When `it` is a `var` (or a `ref` parameter) the loop advances that variable itself, so whatever the iterator records while running is there after the loop and a `break` leaves it where it stopped; over any other expression it iterates a copy.

A loop over a collection that is a variable or field (`for x in xs`, `for k, v in m`, `for x in s.items`) holds the addresses of its elements, so the body may not change the collection: calling a `ref` method on it (`ref xs.push(...)`, `ref m.remove(k)`), passing it by `ref`, or assigning it or anything that holds it (`s = ...` in a loop over `s.items`) is an error, since growing, shrinking or replacing it frees the storage the loop is reading. Writing its elements (`xs[i] = ...`, `for x in ref xs`) is fine. Collect the changes and apply them after the loop, or loop over a copy (`for x in xs.clone()`). Indexes other than two different integer literals count as the same element, so `for x in ll[i]` may not change `ll[j]` either. What a `ref` method of a struct that holds the collection does to it is not checked. Passing by `ref` a struct that holds the collection through a field (`g(ref c)` or `ref c.m()` in a loop over `c.ty.items[i]`) is an error when `g` or `m` may change the collection, as worked out for overlapping arguments above. Calling a function value read from a local, a field or an element (`h.cb()`, `fs[i]()`) whose lambdas change the collection is an error too.

### Slices

A `Slice<T>` is a read-only view into elements owned elsewhere: it indexes, iterates and has `len`, `sub` and the other reading methods, but writing an element through it (`s[i] = v`, `s[i] += 1`, `for x in ref s`, passing `ref s[i]`) is an error, wherever the slice came from (a parameter, a function's result, a struct field). A `MutSlice<T>` is the same view with write access: element writes, `for x in ref s`, `reverseInPlace` and `sortInPlace`, and every `Slice` method besides. `xs.asSlice()` always gives a `Slice`; `xs.asMutSlice()` gives a `MutSlice` and needs a list that may itself be changed (a `var`, a `ref` parameter or one taken by value): on a view parameter, a let binding, a loop or match binding, an element of a `Slice` or a field reached through `view self` it is an error. `sub` keeps the kind it is called on. A `MutSlice<T>` converts to a `Slice<T>` implicitly (as an argument, including to a generic `Slice<T>` parameter, a result, an assigned value or a struct field), never the other way, so a function that only reads takes a `Slice<T>` and one that writes takes a `MutSlice<T>`.

```bismuth
fn clear(MutSlice<int> s) { for x in ref s { x = 0; } }
fn total(Slice<int> s) : int { s.sum() }

var List<int> xs = [3, 1, 2];
xs.asMutSlice().sortInPlace((a, b) => a < b);
clear(xs.asMutSlice().sub(0, 1));
print("{total(xs.asMutSlice())}");   // 5: a MutSlice is a Slice too
```

Until November 2026 a `Slice<T>` could write the elements it viewed, even of a view parameter or a let binding; code that writes through a `Slice` now takes a `MutSlice` (`asMutSlice()`, and `MutSlice<T>` for the parameters, fields and results it writes through).

A slice is a view into elements owned elsewhere, so the same goes for a local slice of a list (`let s = xs.asSlice();`, `xs.asMutSlice()`, `xs.sub(a, b)`, `xs[a..b]`): using it after the list was changed (`ref xs.push(...)`, passing `xs` by `ref`, assigning `xs`), or changing the list inside a loop that uses a slice taken before the loop, is an error; take the slice again after the change, or copy it with `toList()`. Returning a slice of a local list, or of a list parameter taken by value, is an error too, since the list is freed when the function returns. Slices stored in structs or passed to other functions are not tracked for this yet.

### Match

`match` is an expression. Matching on an enum must cover every variant or use `_`. Arms are separated by `,`.

```bismuth
fn area(Shape s) : f32 {
    match s {
        Circle(r)  => PI * r * r,
        Rect(w, h) => w * h,
    }
}

let label = match hp {
    0       => "dead",
    1..=25  => "critical",
    _       => "ok",
};
```

A plain name in a pattern binds the value, unless it names a constant (`MAX`, or `mod.MAX`): then the arm compares against the constant's value, as a literal does.

Pattern matching stays minimal: matching works in `match`, and `let (a, b) = pair;` unpacks tuples, as `for (k, v) in pairs` and `for i, (k, v) in pairs` do for each element (not with `in ref`). There is no nested destructuring in parameters.

## Inheritance and virtual methods

A struct can extend one other struct. It starts with all of the base's fields, at the same offsets, and gets all of its methods. There are no constructors, no access levels beyond `priv`, no multiple inheritance and no hidden allocation: a derived struct is a struct, built with a literal that names the base's fields too.

```bismuth
struct Component {
    int  entity;
    bool enabled = true;

    virtual fn start(ref self) { }               // a hook with a default
    virtual fn update(ref self, f32 dt) { }
    virtual fn name(view self) : str;            // required: no default

    fn describe(view self) : str { "{self.name()}#{self.entity}" }
}

struct Spinner : Component {
    f32 speed = 90.0;
    f32 angle = 0.0;

    fn update(ref self, f32 dt) { self.angle += self.speed * dt; }
    fn name(view self) : str { "spinner" }
}

let s = Spinner { entity: 1, speed: 45.0 };
print(s.describe());                              // spinner#1
```

- **`virtual fn`** marks a method that structs extending this one may replace. A derived struct replaces it by declaring a method with the same name, receiver mode, parameters and result; anything else is an error, and so is declaring a method with the name of one of the base's methods that is not virtual. A virtual method takes `view self` or `ref self`, has no type parameters and is not `async`.
- **Required methods** are virtual methods written with `;` instead of a body. A struct whose own or inherited set of methods still has one can't be built (no literal, no `new`); structs extending it that give it a body can.
- **`super.m(...)`** calls the base's version of `m` from a method of a derived struct, with no dispatch: `ref super.update(dt);`.
- **Passing.** A derived struct is passed by `view` or `ref` wherever its base is expected (`fn draw(view Component c)` takes a `Spinner`), with no copy and no conversion: the base is its prefix. It is not converted to its base by value (`let Component c = spinner;` is an error); a base method taking `self` by copy receives a copy of the base part only.
- **Families of objects.** `Shared<Spinner>` converts to `Shared<Component>` (and `Weak` likewise), so lists of mixed components are `List<Shared<Component>>`. Calling a virtual method through the handle runs the object's own method, and dropping the last handle drops the whole object, its derived fields included, running each level's `drop` from the most derived down.
- **Type tests.** `x is Spinner` is a `bool`; `x as Spinner` turns a `Shared<Component>` (or `Shared<Component>?`) into a `Shared<Spinner>?` that is `null` when the object is something else.
- **Generic bounds.** A struct name is a bound: `fn firstOf<T: Component>(...)` accepts `Component` and the structs extending it.
- **Assignment through a reference** replaces the base part and keeps the object what it is: in `fn reset(ref Component c) { c = ... }` given a `Spinner`, the spinner's own fields are untouched.

A family's virtual methods start in its first struct: a struct extending one that has none can't add any, and once the first has some, every level can add more. A generic struct may extend a plain one (`struct Fire<T> : Hook`): each instance (`Fire<int>`, `Fire<str>`) is a struct of the family, overriding its virtual methods and testable with `is` and `as`, but it can't add virtual methods of its own. Generic structs can't be extended yet, and `packed` structs can't extend or be extended.

### Cost

A struct with virtual methods has a pointer to its type's method table (vtable) in its first 8 bytes, as in C++; one without them has none, so plain data reuse (`struct Named : Point`) costs nothing. A call is direct when the compiler can see that every struct which could be behind the receiver uses the same method; the program is compiled whole, so that is the case for any struct nothing extends, or whose extensions keep the method. Otherwise, when a handful of structs can be behind the receiver, the call compares the object's vtable pointer with theirs and calls the matching method directly (so small methods are inlined); with more, it loads the method from the vtable, and the optimizer turns that back into a direct call wherever it sees which struct an object was built as.

## Errors and nulls

Recoverable errors are values, written `T ! E`, and `?` passes them up. Missing values are `T?`, handled with `?.` and `??`. Neither lets an unhandled case through the compiler.

### Results

`T ! E` is shorthand for `Result<T, E>`. A function returns either a value or an error. `return SomeError;` works because the error type is known.

```bismuth
enum LoadError { NotFound(str path), Empty, BadFormat }

fn loadSprite(str path) : Texture ! LoadError {
    let bytes = fs.read(path)?;            // ? returns the error to the caller
    if bytes.len == 0 { return LoadError.Empty; }
    let image = png.decode(bytes)?;
    Texture.from(image)
}
```

When a returned value would fit either side, as in `str ! str` (or `str? ! str`, or an error type with a `from` variant taking the value's type), it is taken as the value, and bic warns: say which it is with `return .Err("truncated");` or `.Ok(s)`. `.Ok(v)` and `.Err(e)` work wherever a result is expected.

Ways to handle a result:

| Form | Effect |
|---|---|
| `f()?` | Return the error from the current function |
| `f().or(fallback)` | Use fallback on error |
| `f().orElse(e => ...)` | Compute a fallback from the error |
| `f().expect("msg")` | Crash with msg on error, for bugs that should never happen |
| `match f() { Ok(v) => ..., Err(e) => ... }` | Handle both cases explicitly |

`fallback` is an ordinary argument, evaluated before the call whatever the result, so an argument that always leaves the function (`.or({ return e; })`, a `break`) is an error: to leave only on an error, use `?` or a `match` whose `Err` arm returns.

Errors convert automatically only when the caller's error type declares it with `from`:

```bismuth
enum GameError { from LoadError, from NetError, Quit }
```

### Nullables

`T?` must be checked before use. Inside an `if x != null` block, the compiler treats `x` as a plain `T`.
The same holds in a `while x != null` body, in the other branch of `if x == null`, and after an `if x == null { return; }` (or `break`, `continue`) for the rest of the block. Assigning `x` a value that may be null ends this for the rest of the block; such an assignment inside a loop or lambda nested in the check is an error, as the next iteration would use the null. A variable that a lambda assigns is not narrowed, since calling the lambda could null it. Assigning `x` a value that may be null is also an error after a lambda or a `defer` in the checked region reads `x` as non-null, since the lambda could be called, and the `defer` runs, after the assignment; copy what they need into a local first.
`x?.f` is a value, null when `x` is, so it can't be assigned to (`x?.f = v` is an error, as is calling a mutating method through `?.`): check `x` against null and assign `x.f`.
A field path such as `t.target` or `t.a.b` is narrowed the same way when `t` is a local variable or a copy parameter and the narrowed code cannot change `t` (no assignment to `t` or its fields, no `ref t` argument or `ref t.method()` call, no lambda assigning `t`). Fields reached through a `Shared` handle, a `ref` or `view` parameter (including `ref self`) or a pointer are not narrowed, because other code could change them between the check and the use.

```bismuth
let me = players.find(p => p.name == "ryan");   // Player?

print(me?.name ?? "nobody");      // ?. short-circuits, ?? supplies a default
let hp = me?.hp ?? 0;

if me != null {
    print(me.name);                // me is Player here
}

let sure = me!;                    // crash if null; flagged by the linter
```

A `T?` is the value followed by a flag byte (padded to `T`'s alignment),
except for the reference-counted handles `Shared<T>?` and `Weak<T>?`, which
are one word with null as the zero pointer, so a tree of `Shared<Node>?`
children costs no more memory than C pointers.

## Memory

By default, memory is managed with no tracing garbage collector, so there are no collection pauses in a frame. Manual allocation is available but never required.

| Kind | Where it lives | Freed when |
|---|---|---|
| Primitives, vectors, structs | Stack or inline in their container | Scope ends |
| `new T { ... }` | Heap, reference-counted (`Shared<T>`) | Last reference drops |
| `arena.alloc<T>(n)` | An arena you own | `arena.free()` or `arena.reset()` |
| `mem.alloc<T>(n)` | Raw heap | `mem.free(ptr)`, inside `unsafe` |

```bismuth
let tree = new SceneNode { name: "root" };     // Shared<SceneNode>
let weakParent = tree.weak();                   // breaks cycles
```

Copies of a `Shared<T>` are handles to one object. `a == b` on two handles of one type, and `a.ptrEq(b)`, report whether they hold the same object (not equal contents); `ptrEq` also takes a handle to a derived struct (`base.ptrEq(spinner)`), and `Weak<T>` has `ptrEq` too.

The heap is the C library's `malloc` unless the program picks another: `--allocator=bismuth` (std's `BismuthAllocator`, made for threaded programs) or its own struct marked `@allocator` (see [stdlib.md](stdlib.md#allocators)).

### Arenas

Arenas suit per-frame and per-level data: allocate many objects, then free them all in one call.

```bismuth
let frame = Arena(4.mb);
defer frame.free();

var particles = frame.alloc<Particle>(MAX_PARTICLES);   // ArenaSlice<Particle>
// ...
frame.reset();      // reuse the memory next frame, O(1)
```

`alloc` returns an `ArenaSlice<T>`, which indexes and iterates like a `Slice<T>` (its other methods are reached through `asSlice()`, and `asMutSlice()` gives a `MutSlice<T>`). It records the arena's generation, which `reset()` and `free()` advance; using the slice afterwards panics with `arena slice used after reset/free` instead of reading reused or freed memory. The arena's bookkeeping lives outside the freed block, so the check is safe after `free()`. `--unchecked` drops the check, as it drops bounds checks.

### defer

`defer` runs a statement when the current scope exits, on every path, including early returns and `?`.

```bismuth
let file = fs.open(path)?;
defer file.close();
```

### unsafe

Raw pointers (`*T`), pointer arithmetic and `mem.alloc` are allowed only inside `unsafe { }`. Safe code calling an unsafe function must also be wrapped in `unsafe { }`, so the boundary is always visible in the source.

## Concurrency

There are two built-in models: `async`/`await` for waiting on I/O, and `parallel for` for spreading data work across cores. Each has its own threads, one per core by default (`os.setWorkers(n)` or `BISMUTH_WORKERS` sets the count for both): tasks run on the scheduler's worker threads, and `parallel for` splits its iterations over a separate pool of threads, with the thread that reached it taking part.

### async / await

```bismuth
async fn fetchScores(str host) : List<Score> ! NetError {
    let conn = await net.connect(host)?;
    await conn.request<List<Score>>("/scores")?
}

let task = spawn fetchScores("torus:7777");      // runs in the background
let (a, b) = await all(fetchScores("a"), fetchScores("b"))?;
let first  = await race(timeout(2.s), task);
```

An async method that changes its receiver is awaited with the receiver marked: `await ref conn.read()`.

`spawn` returns a `Task<T>`. A task that is never awaited is cancelled when its owning scope exits, so no task can outlive the scope that created it.

Cancelling a task does not unwind it: its waits return early instead. `sleep(d)` returns `false` at once in a cancelled task (`true` when the time passed), so a loop that waits only by sleeping must check it, or `core.taskCancelled()`, to end: `loop { if !sleep(1.s) { break; } poll(); }`. bic warns when a `loop` or `while true` drops `sleep`'s result.

### parallel for

Each iteration runs as a job. The compiler checks that iterations are independent: each one may write only to its own element, and reads of shared data must go through `view`.

```bismuth
parallel for p in ref particles {
    p.vel.y -= 9.8 * dt;
    p.pos   += p.vel * dt;
}

// Nested: one job per tile of the 2D range
parallel for x in 0..img.w, y in 0..img.h {
    img[x, y] = shade(x, y);
}
```

Job granularity can be tuned when profiling shows a need:

```bismuth
parallel(chunk: 256) for i in 0..n { /* ... */ }
parallel(tile: 16x16) for x in 0..w, y in 0..h { /* ... */ }
```

### Reductions

When iterations need to combine results, declare a reduce variable. Each job keeps its own copy, and the copies are merged at the end.

```bismuth
parallel for p in particles reduce (+) alive = 0 {
    if p.life > 0 { alive += 1; }
}
```

### Shared state

For state shared outside these patterns, use `Atomic<T>`, `Mutex<T>`, or channels (`Chan<T>`). The compiler rejects writes to plain shared data from a parallel body or a spawned task.

That includes objects behind a `Shared` handle. When a `Shared<T>` (or `Weak<T>`) is reachable from what a task is given (a spawned call's arguments and receiver, or the variables a parallel body captures; through fields, tuples, nullables, enums, arrays and type arguments such as `List<Shared<T>>`, but not raw pointers), the code the task runs must not write into a `T` through a `Shared` handle: no assignment, `ref` argument, `ref` method call or `for ... in ref` whose place goes through one, in the task's body or in any function it calls. Calling a function value the task was given counts as a write, since its code is unknown. Reading is allowed, and so is writing objects the task creates itself. Not checked: what closures handed to a task capture, and writes by the spawning side while a task reads.

## Modules and namespaces

A file is a module. Modules are named with dotted namespaces, as in C#: each part is a lowercase identifier, and the name maps to a path.

```bismuth
import std.net.http;          // the standard library: std/net/http/
import std.collections as c;  // under another name
import game.world;            // your own: game/world.bi beside this file

fn main() {
    let r = http.get("https://example.com")?;
    let s = c.Set<int>();
    world.load("level1");
}
```

- **Names.** The last part of the namespace is the name the code uses (`http` for `std.net.http`); `as` picks another. Two imports that would give one file the same name are an error until one is renamed with `as`.
- **`std` is the standard library.** Every std module lives under `std`: `std.net`, `std.net.tls`, `std.net.http`, `std.net.ws`, `std.net.url`, `std.net.ssh`, `std.net.ssh.sftp`, `std.net.quic`, `std.net.http3`, `std.io.fs`, `std.io.stdin`, `std.io.poll`, `std.io.compression.lz4`, `std.db.sqlite`, `std.os`, `std.os.process`, `std.os.signal`, `std.os.unix`, `std.os.windows`, `std.os.windows.service`, `std.os.windows.dxgi`, `std.os.windows.d3d11`, `std.os.dbus`, `std.sync`, `std.stream`, `std.time`, `std.cli`, `std.rand`, `std.interop.dl`, `std.interop.objc`, `std.collections`, `std.linq`, `std.window`, `std.window.x11`, `std.gfx`, `std.gfx.image`, `std.gfx.model`, `std.ui`, `std.gamepad`, `std.noise`, `std.collision`, `std.io.compression.deflate`, `std.io.compression.zlib`, `std.io.compression.gzip`, `std.io.archive`, `std.io.archive.zip`, `std.io.archive.tar`, `std.crypto`, `std.crypto.x509`, `std.encoding.hex`, `std.encoding.base64`, `std.encoding.der`, `std.encoding.pem`, `std.uuid`, `std.encoding.json`, `std.audio`, `std.media.h264`, `std.interop.com`, `std.interop.winrt`, `std.encoding.csv`, `std.encoding.toml`, `std.encoding.yaml`, `std.text.regex`, `std.term.vt`. A namespace can both be a module and hold others (`std.net` and `std.net.http`); one that only groups others (`std.io`) can't be imported itself. `std.core` is the prelude and is always visible.
- **Your modules.** Any other name is a module of your program, found relative to the importing program's directory: `game.world` is `game/world.bi`. Directories given with `bic -I dir` are searched after it. Names under `std` are reserved for the standard library, so your files never shadow it.
- **Per-OS files.** When there's no `world.bi`, `import game.world` takes the target's `world_linux.bi`, `world_windows.bi` or `world_darwin.bi`, or `world_unix.bi` on Linux and macOS. A std namespace is the `.bi` files of its directory, and its per-OS files are chosen the same way.
- **Explicit.** A module is visible only through an import: `fs.read(...)` without `import std.io.fs;` is an error that names the import to add.

## Built-ins

### Vector and matrix math

Vectors and matrices are primitive types with operators, swizzles and SIMD-backed storage.

```bismuth
let v = vec3(1, 2, 3);
let w = v * 2.0 + vec3(0, 1, 0);
let flat = v.xz;                    // vec2(1, 3)
let rgba = vec4(v.zyx, 1.0);        // swizzle in constructors
let d = dot(v, w);
let m = mat4.translate(v) * mat4.rotateY(90.deg);
let p = m * vec4(w, 1);
```

`90.deg` and `1.57.rad` both produce an `Angle`. Trig functions take an `Angle`, so degrees and radians cannot be mixed up.

### Printing

`print` takes one interpolated string. Any value can be interpolated, and a format spec goes after `:`.

```bismuth
print("hello");
print("hp {p.hp}/{MAX_HP} at {p.pos:.2}");   // vec3 prints as (1.00, 2.00, 3.00)
print("{player}");                            // whole struct, generated formatting
print("{bytes:hex}  {mask:b}  {ratio:%}");
eprint("warning: {msg}");                     // to stderr
```

### Networking

The `std.net` module needs one line for the common case. Every option has a default and is set in one struct literal.

```bismuth
let server = net.listen(":7777")?;                      // TCP, defaults
let server = net.listen(":7777", {
    proto:      .udp,
    reliable:   true,            // ordered, acked UDP channel
    tickRate:   60,
    maxClients: 32,
    compress:   .lz4,
})?;

let conn = await net.connect("torus:7777")?;
await conn.send(state)?;                             // packed structs go over as raw bytes
for await msg in server.messages<PlayerState>() { /* ... */ }

let page = await net.http.get("https://example.com/api")?;
```

Packed structs are sent as their exact bytes, so there is no serializer to write. Other types use a generated binary encoding. `tickRate` and `codec: .json` are accepted and reserved: neither changes anything yet.

Below the messages are raw byte streams and datagrams, TLS, HTTP and WebSocket. Waits are cooperative: a task blocked on a socket stops when it is cancelled, and every wait can take a timeout.

```bismuth
let s = await net.tls.connect("example.com:443")?;          // or net.tcpConnect for plain TCP
await s.writeText("GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n")?;
let status = await s.readLine()?;

var app = http.Server.new();
ref app.get("/players/:id", req => http.Response.json(200, lookupPlayer(req.param("id"))));
ref app.websocket("/live", (http.Request req, ws.Socket sock) => { /* ... */ });
spawn app.listen(":8080");

let sock = await ws.connect("wss://torus:8080/live")?;
let ips = await net.lookup("torus")?;                         // List<str>, IPv4 and IPv6
```

### Files embedded at compile time

`core.includeStr("path")` is a file's text as a `str`, read when the program is compiled, so the program needs no file at run time. The path is a string literal, relative to the source file holding the call (or absolute); the file must be UTF-8 text. It is a constant, so it can initialise a `const`:

```bismuth
const SCHEMA = core.includeStr("sql/schema.sql");   // beside this .bi file
```

### Hot reload

Mark a function `@hot` and you can edit it while the program runs. Running state is kept. Changing the layout of a struct that is in use triggers a migration hook if you define one, and a full restart if you don't.

```bismuth
@hot
fn updateParticles(ref ArenaSlice<Particle> ps, f32 dt) { /* edit me live */ }

@hot(onReload: rebuildCaches)
fn loadLevel(str name) : Level ! LoadError { /* ... */ }
```

### C interop

C libraries are called through `extern c fn` declarations. `bic bindgen` writes them from a header, and the result is checked in and imported like any module, so building needs no C compiler:

```bismuth
// gfx.bi, from `bic bindgen gfx.h -o gfx.bi`
struct Color { u8 r; u8 g; u8 b; u8 a; }
const RED = Color { r: 230, g: 41, b: 55, a: 255 };
@link("gfx") extern c fn InitWindow(i32 width, i32 height, str title);
```

```bismuth
import gfx as rl;

rl.InitWindow(1280, 720, "bismuth");

export c fn bismuth_tick(f32 dt) { /* callable from C */ }
```

- **Types** map by size on the target: `int` is `i32`, `float` is `f32`, and `long` is `i64` on Linux and macOS but `i32` on Windows. Bindings that differ by target go in `gfx_linux.bi`, `gfx_windows.bi` and `gfx_darwin.bi` (or `gfx_unix.bi`), and `import gfx` picks the target's file when there is no `gfx.bi`.
- **Libraries** are named without prefix or extension: `@link("gfx")` links `libgfx.so` on Linux, `gfx.dll` on Windows and `libgfx.dylib` on macOS. Without `@link`, a function comes from the C library. `@cname("c_name")` binds a Bismuth name to a different C symbol, for an `extern c fn` and for an `export c fn` alike. `@link(static: "libgfx.a")` links an archive or object file (relative to the declaring file) into the executable instead, and `@link(static: "sqlite3")` a C library bic builds from `third_party/` (Linux so far; see [c-interop.md](c-interop.md#static-c-libraries)).
- **Strings.** A `str` argument is passed as a `const char*` to its NUL-terminated bytes.
- **Pointers.** C pointers come in as `*T`. An `unsafe extern c fn` can be called only inside `unsafe`; bindgen marks functions that take or return raw pointers that way, except those a header marks `@bi.safe`.
- **Structs by value.** `extern c fn`s, C function pointers and `export c fn`s take and return structs by value with the target's C convention: on x86-64, SysV on Linux and macOS and the Microsoft x64 convention on Windows (structs of 1, 2, 4 or 8 bytes in a register, others as a pointer to a copy; results over 8 bytes through a hidden pointer). An `export c fn`'s parameters and result cannot own memory (`str`, `List<T>`, ...), since C copies them bitwise. Bismuth calls an `export c fn` with the same convention C does.
- **Generated files.** A file whose first line is `// Code generated ... DO NOT EDIT.` is not linted for naming, since its names are C's.

`import c` reads a header directly, as a shortcut while exploring a library:

```bismuth
import c "raylib.h" as rl link "raylib";
```

It runs the C preprocessor (`$CC`, else `cc`) at every build, with the target's type sizes and OS macros, so it needs a C compiler on every build machine. `link` takes a library name, or a shared library file of the target's kind. Static archives and object files are not supported. See [c-interop.md](c-interop.md) for the design and how it compares with other languages.

A C function pointer has a type of its own, `extern c fn(i32, *u8) : i32`. A value of that type is made from an address with `core.bitcast` (from `dl` lookups, COM vtables or `core.fnAddr` of an `export c fn`) and called like a function inside `unsafe`, with the target's C calling convention:

```bismuth
type Strlen = extern c fn(*u8) : u64;
let strlen = unsafe { core.bitcast<Strlen>(dl.c().sym("strlen")) };
let n = unsafe { strlen("bismuth".cstr()) };
```

A C function pointer type can end in `...` for a variadic C function, as an `extern c fn` declaration can. The arguments after the fixed ones get C's default promotions (`f32` to `f64`, integers narrower than 32 bits to `i32`, a `str` to its `const char*`) and are passed as the target's convention wants variadic arguments: on x86-64 SysV the caller sets AL to the number of vector registers used, on Win64 a variadic float also goes in its integer register, and on arm64 Linux (AAPCS64) they travel like fixed arguments. Apple's arm64 ABI, which puts every variadic argument on the stack, is not supported (bic has no Mach-O arm64 target yet). Only `extern c fn` types can be variadic.

```bismuth
type Snprintf = extern c fn(*u8, u64, *u8, ...) : i32;
let snprintf = unsafe { core.bitcast<Snprintf>(dl.c().sym("snprintf")) };
unsafe { snprintf(buf, 64, "%d %.2f %s".cstr(), i32(42), 1.5, "text"); }
```

A COM method, a C++ member function reached through an object's vtable, has the type `extern com fn(*T, ...) : R`. Its first parameter is the object (`this`) and must be a pointer. Arguments and scalar results go as for `extern c fn`, but on Windows a struct result of any size comes back through a hidden pointer passed after `this`, which is MSVC's rule for member functions: a C function returns a struct of 8 bytes or less in RAX, and passes the pointer for a larger one first. On other targets a member function returns like a C function (the Itanium C++ ABI), so there `extern com fn` is called as `extern c fn` is:

```bismuth
struct CpuHandle { u64 ptr; }
// ID3D12DescriptorHeap::GetCPUDescriptorHandleForHeapStart, vtable slot 9
type GetCpuStart = extern com fn(*u8) : CpuHandle;
let vtbl = unsafe { mem.cast<*u8>(mem.cast<*u8>(heap)[0]) };
let start = unsafe { core.bitcast<GetCpuStart>(vtbl[9])(heap) };
```

A method written in Bismuth for C++ to call (an object Bismuth implements) is an `export c fn` with the hidden pointer spelled out: on Windows, `export c fn getDesc(*Obj obj, *Desc out) : *Desc` stores to `out[0]` and returns `out`.

### Shader stages

A function marked `@vertex`, `@fragment` or `@compute(x, y, z)` is a GPU shader stage, written in a subset of the language and compiled by bic to SPIR-V, MSL and HLSL. Its parameter and result types are its interface, so vertex layouts and bindings come from ordinary structs, and it stays an ordinary function that CPU code can call. `core.shader(f)` gives the compiled stage as a `ShaderCode` value, built at compile time. See [shaders.md](shaders.md).

```bismuth
struct Vertex { vec2 pos; vec4 color; }                 // a vertex buffer: one attribute per field
struct Varyings { @position vec4 pos; vec4 color; }     // the vertex stage's result
struct Transform { mat4 mvp; }

@vertex
fn colorVs(Vertex v, @uniform Transform t) : Varyings {
    Varyings { pos: t.mvp * vec4(v.pos, 0.0, 1.0), color: v.color }
}

@fragment
fn colorFs(Varyings i) : vec4 { i.color }

let vs = core.shader(colorVs);   // vs.spirv, vs.msl, vs.hlsl, vs.attrs, vs.bindings
```

- **Parameters.** A struct is a vertex buffer (vertex stage) or the varyings (fragment stage); `@uniform T` is a uniform block; `Texture2D` and `Sampler` are texture and sampler bindings; `MutSlice<T>` is a read-write storage buffer and `Slice<T>` (or `view Slice<T>`) a read-only one; `@vertexIndex`, `@instanceIndex`, `@frontFacing`, `@globalId`, `@localId`, `@groupId` and `@localIndex` are system values. `@instance` marks a per-instance vertex struct.
- **Fields** of interface structs take `@position` (clip-space position, and the fragment's window position on input), `@depth` (fragment depth output) and `@flat`.
- **The subset**: `bool`, `i32`, `u32`, `f32`, vectors, `mat4`, structs, fixed arrays, plain enums, `if`, loops, helper functions and methods. Not strings, the heap, closures, nullables, results, pointers, `f64`, recursion or `match` (yet). `int` is computed in 32 bits on the GPU. Each violation is a compile error at the expression.

## Full example

This small multiplayer particle demo uses every feature above: C interop, packed network structs, ranged types, ref passing, results, nullables, an arena, nested parallel-for, async networking and hot reload.

```bismuth
import raylib as rl;   // raylib.bi: bindings from `bic bindgen raylib.h`
import std.net;
import std.io.fs;
import png;            // png.bi beside this file

const MAX_PARTICLES = 4096;
const TICK_RATE     = 60;

type Hp = int in 0..=100;
distinct type EntityId = u16;

enum Team { Red, Blue, Spectator }

// Exact bytes on the wire: no padding, big-endian
struct PlayerState packed(be) {
    EntityId id;
    vec3     pos;
}
comptime assert sizeof(PlayerState) == 14;

struct Particle packed {
    vec3 pos;
    vec3 vel;
    f32  life;
}

struct Player {
    EntityId  id;
    str       name;
    vec3      pos;
    Team      team;
    Hp        hp     = 100;
    EntityId? target = null;
}

impl Player {
    fn takeHit(ref self, int dmg) {
        self.hp = Hp.clamp(self.hp - dmg);
    }
}

fn lerp(f32 a, f32 b, f32 t) : f32 { a + (b - a) * t }

fn teamColor(Team t) : rl.Color {
    match t {
        Red       => rl.RED,
        Blue      => rl.BLUE,
        Spectator => rl.GRAY,
    }
}

fn loadSprite(str path) : rl.Texture ! LoadError {
    let bytes = fs.read(path)?;
    let image = png.decode(bytes)?;
    rl.LoadTextureFromImage(image)
}

@hot
fn updateParticles(ref ArenaSlice<Particle> ps, f32 dt) {
    parallel for p in ref ps {
        p.vel.y -= 9.8 * dt;
        p.pos   += p.vel * dt;
        p.life  -= dt;
    }
}

fn bakeLightmap(ref Image img, vec3 sun) {
    let s = sun.normalized;
    parallel for x in 0..img.w, y in 0..img.h {
        img[x, y] = Color.gray(max(dot(img.normalAt(x, y), s), 0.0));
    }
}

async fn runServer() : void ! NetError {
    let server = net.listen(":7777", { proto: .udp, tickRate: TICK_RATE })?;
    for await msg in server.messages<PlayerState>() {
        print("player {msg.id} at {msg.pos.xz}");
        server.broadcast(msg)?;
    }
}

fn main() : void ! GameError {
    rl.InitWindow(1280, 720, "bismuth");
    defer rl.CloseWindow();

    let arena = Arena(MAX_PARTICLES * sizeof(Particle));
    defer arena.free();
    var particles = arena.alloc<Particle>(MAX_PARTICLES);

    spawn runServer();

    let flame = loadSprite("assets/flame.png").or(rl.LoadDefaultTexture());
    var players = List<Player>();
    let me = players.find(p => p.name == "ryan");

    while !rl.WindowShouldClose() {
        let dt = rl.GetFrameTime();
        updateParticles(ref particles, dt);

        let alive = particles.asSlice().filter(p => p.life > 0).count();
        let target = me?.target?.str() ?? "none";
        print("target: {target}  particles: {alive}");
    }
}
```

## Open questions

- [ ] **Block values.** A block's value is its last expression without `;`, as in Rust. One stray `;` changes the type to `void`. Should we require `return` everywhere instead?
- [ ] **Mutating method calls.** The current syntax is `ref player.takeHit(25);`. Alternatives are `player.takeHit!(25);` or no marker for methods, relying on the receiver being a `var`.
- [ ] **Variable syntax.** `let i32 a` keeps types first as in parameters, but `let a: i32` is more familiar. Which reads better?
- [ ] **Units of measure.** Only `Angle` and `Duration` are built in. Should `m` and `m/s` become checked types like them?
- [ ] **Contracts.** The survey left out `expect`/`ensure`. Ranged types cover part of that need. Are invariant checks wanted too?
- [ ] **Default alignment.** Should plain structs stay naturally aligned, or should `packed` be the default with an aligned opt-out?
- [ ] **Interfaces.** A struct has one base. Should there be field-less interfaces a struct can take several of (dispatched through a second pointer, as Go's interfaces are), or do bases and generic bounds cover enough?
- [ ] **Hot reload scope.** Should `@hot` be per function, per module, or the whole program in dev builds?
