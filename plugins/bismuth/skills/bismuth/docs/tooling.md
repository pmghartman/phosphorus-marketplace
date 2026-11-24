# Bismuth tooling

The toolchain is two binaries built from this repository:

| Binary | What |
|---|---|
| `bic` | compiler (`build`, `run`, `check`, `ir`), shader compiler (`shader`), C binding generator (`bindgen`), TypeScript declarations (`ts`), formatter (`fmt`) and language server (`lsp`) |
| `bils` | the language server on its own, for editors that expect a dedicated binary |

See [Installing](#installing) for how to put them on a machine.

## Installing

There is one supported way to install, update and remove the toolchain:
`bic install`, `bic upgrade` and `bic uninstall`. The install scripts are a
way to run the first `bic install` on a machine that has no bic yet.

```sh
# Linux, macOS
curl -fsSL https://git.philab.io/ryan/bismuth/raw/main/scripts/install.sh | sh
# Windows, PowerShell
irm https://git.philab.io/ryan/bismuth/raw/main/scripts/install.ps1 | iex
```

Each script finds the newest release on
<https://git.philab.io/ryan/bismuth/releases>, downloads the package for
the machine (`bic-VERSION-<platform>.tar.gz`, or `.zip` for Windows),
checks it against the release's `SHA256SUMS` and refuses it on a mismatch,
clears macOS's quarantine attribute, and runs the `bic install` inside it,
which also puts the package's Claude Code skill for Bismuth in
`~/.claude/skills/bismuth` (see [below](#the-claude-code-skill)).
`install.sh` takes `bic install` flags after `sh -s --`; `install.ps1` takes
them as arguments when run as a file. Both are also attached to every
release (and listed in its `SHA256SUMS`).

| Variable | |
|---|---|
| `BISMUTH_VERSION` | install that release (`0.1.30`) instead of the latest |
| `BISMUTH_RELEASES_API` | the releases list (JSON), default `https://git.philab.io/api/v1/repos/ryan/bismuth/releases` |
| `BISMUTH_DOWNLOAD` | the base of release files, at `<base>/<tag>/<file>`; default `https://git.philab.io/ryan/bismuth/releases/download` |
| `BISMUTH_NO_CLAUDE_SKILL` | `1`: don't install the [Claude Code skill](#the-claude-code-skill) |

`BISMUTH_RELEASES_API` and `BISMUTH_DOWNLOAD` point at a mirror;
`file://` URLs work. `bic upgrade` reads the same variables.

### bic install

`bic install` copies the running `bic`, and `bils` when it sits in the same
directory (every release package has both), into the bin directory:

| | Per user (default) | `--system` |
|---|---|---|
| Linux | `~/.local/bin` | `/usr/local/bin` |
| macOS | `~/.local/bin` | `/usr/local/bin` |
| Windows | `%LOCALAPPDATA%\Programs\Bismuth\bin` | `%ProgramFiles%\Bismuth\bin` |

The package's `third_party/` (the C sources of the libraries bic builds
itself, such as SQLite for `sqlite.useBundled()`) goes into
`~/.local/share/bismuth/third_party` (`/usr/local/share/bismuth/third_party`
with `--system`, `DIR/share/bismuth/third_party` with `--prefix`;
`third_party` beside `bin` on Windows), where the installed bic finds it;
`bic uninstall` removes it.

`--prefix DIR` installs into `DIR/bin` instead. The per-user install needs
no root or administrator rights. `~/.local/bin` is the XDG location that
systemd's file hierarchy, pipx and many distributions already put on PATH;
on macOS it is used too rather than Homebrew's `/opt/homebrew/bin`, which
belongs to Homebrew.

Running `bic install` again is always safe, and is how an install is
updated: a binary with the same bytes is left alone, a different one is
replaced atomically (written to a temporary file beside it, then renamed
over it), so the installed `bic` is always a whole old one or a whole new
one. On Windows the running `bic.exe` can't be overwritten or deleted, but
it can be renamed: it is moved aside to `bic.exe.old`, which the next
install removes. `install.json` in the state directory records the
installed version, which `bic version` shows:

```
$ bic version
bic 0.1.31
installed: bic 0.1.31 in /home/you/.local/bin
profiles: default -O2, --debug -O1 -g, --release -O2
```

What `bic install` prints is what it changed. Its flags (`uninstall` and
`upgrade` take the same):

| Flag | |
|---|---|
| `--system` | install machine-wide (above); needs root or an administrator |
| `--prefix DIR` | install into `DIR/bin` |
| `--no-modify-path` | leave PATH and shell profiles alone |
| `--no-register` | leave `.bi` file types alone |
| `--no-claude-skill` | leave out the Claude Code skill (below); so does `BISMUTH_NO_CLAUDE_SKILL=1` |
| `--registry-root KEY` | Windows: write `HKCU\KEY\Environment` and `HKCU\KEY\Classes` instead of the real keys, and broadcast nothing; for tests |

#### PATH

Linux and macOS work as rustup does. When the bin directory is not on
PATH already, `bic install` writes `~/.local/share/bismuth/env`, a POSIX
shell script that prepends the directory unless PATH already holds it,
and adds one line, `. "$HOME/.local/share/bismuth/env"`, to `~/.profile`,
to each of `~/.bash_profile`, `~/.bash_login` and `~/.bashrc` that exists,
and to `~/.zshenv` (`$ZDOTDIR/.zshenv`) when zsh is in use, which is always
on macOS. fish gets `~/.config/fish/conf.d/bismuth.fish` when fish is
installed. A profile that already has the line is left alone, and the env
script checks PATH before it adds to it, so the directory is on PATH once
however often either runs. When the directory already is on PATH (many
distributions add `~/.local/bin` once it exists), no profile is touched.
`--system` never edits profiles: `/usr/local/bin` is on PATH by default.

Windows works as the Go, Node and rustup installers do: the bin directory
is appended to the user `Path` value in `HKCU\Environment` (the machine
`Path` in `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Environment`
with `--system`) unless an entry already names it (compared without case,
trailing `\` or `%VARIABLE%` differences), keeping the value's
`REG_EXPAND_SZ` type, and `WM_SETTINGCHANGE` is broadcast so terminals
opened afterwards see it. Terminals already open keep their old PATH,
except the one that ran `install.ps1`: the script adds the bin directory
to that session's `$env:Path` too (not with `--no-modify-path`).

#### File types

- **Linux**: a shared-mime-info package,
  `~/.local/share/mime/packages/bismuth.xml`, declares `text/x-bismuth`
  for `*.bi`, a sub-class of `text/plain`, so file managers show a Bismuth
  icon (`~/.local/share/icons/hicolor/scalable/mimetypes/text-x-bismuth.svg`)
  and offer text editors to open it. `update-mime-database` then rebuilds
  the cache; without shared-mime-info installed, the package is picked up
  once it is.
- **Windows**: `HKCU\Software\Classes\.bi` points to the ProgID
  `Bismuth.Source` (with `Content Type` `text/x-bismuth`, `PerceivedType`
  `text`, and an `OpenWithProgids` entry), which has the Bismuth icon
  (`%LOCALAPPDATA%\Programs\Bismuth\bismuth.ico`), **Open** and **Edit**
  in Notepad, and **Run with bic**, which runs the file in a console that
  stays open. If `.bi` files already open with another program (an
  editor registered its own ProgID), that default is kept and
  `Bismuth.Source` is only added under **Open with**; a choice made in
  Explorer ("Always use this app") overrides all of this, as it should.
  Explorer is told to reread associations (`SHChangeNotify`).
- **macOS**: nothing is registered; see below.

##### File types on macOS

macOS identifies file types by Uniform Type Identifier, and only an app
bundle can declare one (`UTExportedTypeDeclarations` in its `Info.plist`,
read by Launch Services when it finds the bundle). A command-line tool in
`~/.local/bin` can't. The options were:

1. **A tiny `Bismuth.app`** in `~/Applications` that declares a UTI for
   `.bi` (conforming to `public.source-code` and `public.plain-text`) and
   does nothing else. It would give `.bi` files a name and an icon in
   Finder, but it shows up in Spotlight and Launchpad as an app that does
   nothing when opened; unsigned and not notarized, it gets Gatekeeper
   warnings; Launch Services keeps its registration after the bundle is
   deleted until `lsregister -u` runs; and it still wouldn't choose an
   editor, since the type's handler is whatever app declares it can open
   it.
2. **Leave it to editors.** macOS gives an undeclared extension a dynamic
   UTI; Finder's **Open With** offers any app, **Always Open With** sticks
   for the extension, and editors that support Bismuth (the extensions in
   [`editors/`](../editors)) take care of highlighting and bils.

`bic install` does the second, which is also what Go, rustup, Deno, Bun and
Zig do on macOS for their source files: none of them registers a file type.
It changes nothing on the system that the user would have to find and
undo, and an editor that declares a Bismuth UTI later (an editor bundle
can) takes over cleanly. A dedicated `.app` is worth it only once there is
a Bismuth GUI application to own the type.

#### The Claude Code skill

Every release package carries a [Claude Code](https://claude.com/claude-code)
skill for Bismuth in `skills/bismuth`: a `SKILL.md` that teaches the
language (syntax, types, `ref` and `view`, results and nullables, the
std modules, bic's commands and the usual mistakes) and loads whenever
Claude reads or writes `.bi` files, plus these docs for it to read for
more. Its source is
[`tools/claude-skill/bismuth/SKILL.md`](../tools/claude-skill/bismuth/SKILL.md),
and every code example in it compiles.

`bic install` copies that folder to `~/.claude/skills/bismuth`
(`%USERPROFILE%\.claude\skills\bismuth` on Windows, or
`$CLAUDE_CONFIG_DIR/skills/bismuth` when that is set), so Claude Code
knows the language without being told, and records it in `install.json`;
`bic upgrade` updates it with the rest, dropping files a newer release no
longer has, and `bic uninstall` removes it. A `bic` with no skill beside
it (one built from source) leaves an installed skill as it is.
`--no-claude-skill` or `BISMUTH_NO_CLAUDE_SKILL=1` (which the install
scripts pass on) leaves it out and takes off one installed before.
`--system` installs skip it, since skills are per user. To install it by
hand, copy `tools/claude-skill/bismuth` (and `docs/` into it) to
`~/.claude/skills/`.

Each release also publishes the skill, with the docs, as the `bismuth`
plugin of the Claude plugin marketplace at
`https://github.com/pmghartman/phosphorus-plugin`
(`tools/ci/publish-plugin.sh`), for Claude apps that don't run
`bic install`. In Claude Code: `/plugin marketplace add
https://github.com/pmghartman/phosphorus-plugin`, then `/plugin install
bismuth@phosphorus`, with auto-update on for the marketplace; in
claude.ai, Claude Desktop or Cowork: Customize, Plugins, Add marketplace,
then install Bismuth. With both the plugin and `bic install`'s copy,
Claude Code loads the skill twice: pass `--no-claude-skill` to keep only
the plugin.

### bic upgrade

`bic upgrade` installs the latest release when it is newer than the
installed version (`--version V` picks one, `--force` reinstalls or
downgrades): it downloads the package for this machine and `SHA256SUMS`,
refuses the package unless its SHA-256 matches, unpacks it and runs the new
`bic install` with the same flags, so the new version installs itself. On
Windows that replaces the running `bic.exe` as above.

### bic uninstall

`bic uninstall` removes `bic` and `bils` from the bin directory, the PATH
entry (the profile lines and env script, or the `Path` entry), the file
type, the icon, the Claude Code skill it installed and `install.json`. Shared directories such as
`~/.local/bin` stay; on Windows the `Bismuth` directory goes once it is
empty. A profile that held nothing but the Bismuth line is removed; any
other keeps the rest of its contents. The running `bic.exe` on Windows is
moved to the temporary directory and deleted once it exits.

### Building from source

bic is written in Bismuth and built by a bic. From a checkout,
`tools/ci/bootstrap.sh` builds it from the latest released bic (stage 1,
then stages 2 and 3, which must match); build bils beside it with the
new bic and install from there:

```sh
tools/ci/bootstrap.sh out/bic
(cd compiler && ../out/bic build --no-lint -o ../out/bils bils.bi)
out/bic install
```

With a bic already installed, `compiler/build.sh bic out/bic` is one
build instead of three. [compiler/README.md](../compiler/README.md) has
the details.

### Tests

The compiler's unit tests (`compiler/tests/tool_install.bi`) install,
reinstall, update and uninstall into temporary homes and prefixes, with
the Windows logic run against an in-memory registry on every OS (and, on
Windows, the real registry under a throwaway key). CI runs [`tools/ci/test-install.sh`](../tools/ci/test-install.sh)
on Linux (and macOS) and [`tools/ci/test-install.ps1`](../tools/ci/test-install.ps1)
on Windows against each build's release packages: the install script,
`bic install` from the package and from the installed bic, `bic upgrade`
(up to date, forced over the running bic, and with a bad checksum), and
`bic uninstall`, checking each time that PATH holds the bin directory once
and `.bi` has one association. On Windows they run under a temporary
`%LOCALAPPDATA%` with `--registry-root`, and check that the host's own
`Path` and file types are unchanged afterwards.

## Shared libraries and import paths

`bic build --library file.bi` writes a shared library instead of an
executable: `libfile.so` on Linux (x86-64 or arm64) and `file.dll` on
Windows; macOS `.dylib` output is not supported yet. A library needs no
`main`. Its `export c fn`s are its exported symbols (`@cname("c_name")`
picks a symbol name other than the function's), callable from C, from any
language's foreign function interface, or from a Bismuth program through
`std.interop.dl`:

```bismuth
// game.bi: bic build --library -o build/libgame.so game.bi
@cname("game_tick")
export c fn gameTick(f32 dt) : i32 { ... }

// host.bi
import std.interop.dl;

fn main() {
    unsafe {
        let lib = dl.open("build/libgame.so") ?? panic("no game library");
        let tick = core.bitcast<extern c fn(f32) : i32>(lib.sym("game_tick"));
        tick(0.016);
    }
}
```

A library carries its own copy of the runtime and the standard library.
The dynamic loader sets it up when it loads the library (`DT_INIT_ARRAY`
on Linux, `DllMain` on Windows) and flushes the library's buffered
standard output when it is unloaded or the process exits. Two Bismuth
images in one process (a program and the libraries it loads, or several
libraries) therefore share nothing but the C library: each has its own
standard output buffer, scheduler and `core.static` storage. They can pass
each other plain data, `export c fn` pointers and memory from the C
library's heap (the default allocator); a Bismuth object made by one image
should be used through functions of that image, since a struct's methods,
its virtual methods' table and its `drop` are compiled into the image that
made it.

`-I dir` (repeatable) adds a directory where `import` finds a program's own
modules, searched after the main file's directory, so a project can use
modules that live elsewhere (an engine's SDK, a shared library of code):

```sh
bic build --library -I ../engine -o build/libgame.so game.bi   # import engine.api;
```

A project of several programs and libraries is built with
[bimake](bimake.md) (`bic make`), which reads a `BiMakeLists.txt` in
CMake's language and runs bic for each target with these flags:

```cmake
project(game)
add_library(game SHARED game.bi)
target_include_directories(game PRIVATE ../engine)
add_executable(host host.bi)
target_link_libraries(host PRIVATE game)
```

## GUI programs: icons, manifests and app bundles

```sh
bic build --gui --icon icon.png --app-version 1.2.0 app.bi          # Windows: app.exe with resources
bic build --app --icon icon.png --app-id com.example.app app.bi     # macOS: app.app
bic build --app --icon icon.png --app-name "My App" --target macos app.bi   # from any OS
```

| Flag | Windows | macOS (with `--app`) |
|---|---|---|
| `--app` | the same as `--gui` | write `Name.app` instead of a bare executable |
| `--icon file` | `RT_ICON` and `RT_GROUP_ICON` resources | `Contents/Resources/AppIcon.icns` |
| `--app-name name` | `ProductName`, `FileDescription`, `InternalName` (default: the executable's name) | `CFBundleName`, `CFBundleDisplayName`, the bundle's and executable's file name (default: `-o` or the source file's name) |
| `--app-id id` | (not used) | `CFBundleIdentifier` (default `local.bismuth.<name>`; set your own reverse-DNS id for anything you ship) |
| `--app-version v` | `FileVersion`, `ProductVersion` and the fixed version (up to four numbers) | `CFBundleShortVersionString`, `CFBundleVersion` (default `1.0`) |
| `--company name` | `CompanyName` | (not used) |
| `--copyright text` | `LegalCopyright` | `NSHumanReadableCopyright` |
| `--manifest file` | embed this manifest instead of the default | |
| `--sign-identity id` | | on a Mac, sign with `codesign` for distribution (below) |

The flags are an error for Linux targets, and for macOS without `--app`:
a bare macOS executable has nowhere to keep an icon.

### Icons

`--icon` takes a PNG or a `.ico` file. A PNG should be square and
1024×1024, which gives every size both platforms use; a smaller one gives
only the sizes up to its own (images are never enlarged), and one that
isn't square is centred on a transparent square. The other sizes are
made by shrinking with a box filter: each output pixel is the
area-weighted average of the source pixels it covers, in premultiplied
alpha. Design small sizes by hand and pass a `.ico` if the shrunk ones
are too soft.

- **Windows** gets 16, 20, 24, 32, 40, 48 and 64 pixels as 32-bit BGRA
  bitmaps and 256 as a PNG. A `.ico` is embedded image for image, as it
  is.
- **macOS** gets a `.icns` of PNG entries: `icp4`, `icp5`, `ic07`,
  `ic08`, `ic09` (16, 32, 128, 256, 512 points) and `ic11` to `ic14` and
  `ic10` (the same at 2x, up to 1024 pixels). From a `.ico`, the largest
  PNG or 32-bit image is the source.

[`compiler/assets/bismuth-app.png`](../compiler/assets/bismuth-app.png)
is the Bismuth app icon; CI builds `examples/window` with it.

### Windows resources

Every Windows executable has a `.rsrc` section with an application
manifest. That changes no program's behaviour beyond these settings:

| Manifest setting | Programs | Why |
|---|---|---|
| `activeCodePage` UTF-8 | all | the C runtime's narrow `argv`, environment and file names (`_open`, `FindFirstFileA`), which std uses, become UTF-8 as on Linux and macOS, instead of the system's ANSI code page (Windows 10 1903 and later) |
| `longPathAware` | all | paths past 260 characters, where the system allows them |
| `supportedOS` Windows 10 | all | Windows 10 behaviour rather than compatibility shims |
| `requestedExecutionLevel asInvoker` | all | Windows never guesses from the name (`setup.exe`) that it needs elevation |
| `dpiAwareness` PerMonitorV2 | `--gui` | sizes in physical pixels, no bitmap stretching; std's window already asks for this at run time, the manifest makes it apply before any window exists |
| Common Controls 6.0 | `--gui` | themed message boxes and dialogs |

`--manifest` replaces it whole (for `requireAdministrator`, say). A GUI
program (`--gui` or `--app`) also gets version information
(`VS_VERSIONINFO`, the Details tab in a file's Properties), and so does
any program built with `--app-name`, `--app-version`, `--company` or
`--copyright`. `--icon` adds the icon: Explorer and the taskbar show it,
and std's `window` registers its window class with it (`LoadIconW` of
group icon 1), so it is the title bar's too. Check with
`llvm-readobj --coff-resources app.exe`.

### macOS app bundles

```
Name.app/Contents/Info.plist
Name.app/Contents/MacOS/Name                 the executable
Name.app/Contents/Resources/AppIcon.icns     with --icon
Name.app/Contents/_CodeSignature/CodeResources
```

`Info.plist` has the bundle id, name, version, `CFBundleIconFile`,
`NSHighResolutionCapable` (Retina resolution rather than scaled-up
points), `NSSupportsAutomaticGraphicsSwitching` (the integrated GPU is
fine for Metal), `NSPrincipalClass` `NSApplication` and
`LSMinimumSystemVersion` 10.15. Building over an existing bundle replaces
it; bic refuses to replace anything at that path that isn't one.

The bundle is signed ad hoc, as `codesign --sign - Name.app` would sign
it: the executable's signature (a SHA-256 hash per 4 KB page, as for every
macOS executable bic writes) names the bundle id and binds the bundle in
through the CodeDirectory's special slots, the hashes of `Info.plist`
(slot 1), of the empty requirement set (2) and of
`_CodeSignature/CodeResources` (3), which lists the SHA-1 and SHA-256 of
every resource under codesign's default rules. Changing any file in the
bundle breaks the signature. An ad-hoc signature is enough to run anywhere the bundle isn't
quarantined (Apple Silicon requires at least that) and passes
`codesign --verify --strict --verbose=2 Name.app`, but Gatekeeper rejects
it in a download: a quarantined app needs Developer ID and notarization.

### Developer ID signing and notarization

On a Mac with a "Developer ID Application" certificate in the keychain,
`--sign-identity` re-signs the bundle with it and the hardened runtime,
which notarization requires (`codesign --force --timestamp --options
runtime --sign ID Name.app`). Then submit, wait and staple:

```sh
bic build --app --icon icon.png --app-id com.example.app \
  --sign-identity "Developer ID Application: Example Ltd (TEAMID1234)" app.bi
ditto -c -k --keepParent app.app app.zip
xcrun notarytool submit app.zip --keychain-profile notary --wait
xcrun stapler staple app.app            # the ticket, for offline Gatekeeper checks
spctl --assess --type execute --verbose app.app
```

`xcrun notarytool store-credentials notary` saves the Apple ID, team id and
app-specific password once. Ship the stapled `app.app` in a zip or a
disk image (`hdiutil create -srcfolder app.app app.dmg`, then sign,
notarize and staple the `.dmg` the same way). The hardened runtime
allows what std does (`dlopen` of system libraries, Objective-C classes
made at run time); a program that loads unsigned third-party dylibs
(MoltenVK from Homebrew, say) needs the
`com.apple.security.cs.disable-library-validation` entitlement: sign it
with `codesign --entitlements` by hand. `--sign-identity` needs macOS;
elsewhere build the bundle and sign it on a Mac.

## bic shader

`bic shader [-o dir] [--entry fn] file.bi` writes each shader stage in a
program (or only `fn`) to `dir` as `<fn>.bishader`, a package
`gfx.loadShader` reads, and as `.spv`, `.metal` and `.hlsl`, with `.dxbc`
on Windows and `.metallib` on a Mac with Xcode. `bic build` and `bic run`
put the same binaries in the program's `ShaderCode`s when the host can
make them for the target, cached under the user's cache folder
(`bismuth/shaders`); `BIC_SHADER_BINARIES=0` turns that off. See
[shaders.md §3.1](shaders.md#31-precompiled-binaries).

## bic bindgen

```sh
bic bindgen gfx.h -o gfx.bi           # bindings for Linux, Windows and macOS
bic bindgen --lib raylib -I include raylib.h -o raylib.bi
bic bindgen --target windows gfx.h    # one target, to stdout
```

Writes a C header's functions, structs, enums and macros as Bismuth
declarations (`extern c fn`, `struct`, `type`, `const`) to check in, so
building needs no C compiler (generating does: it runs `$CC`, else `cc`).
Functions are linked from the library `--lib` names, by default the
header's name (`gfx` links `libgfx.so`, `gfx.dll` or `libgfx.dylib`).
When the declarations differ between targets, each gets its own file
(`gfx_linux.bi`, `gfx_windows.bi`, `gfx_darwin.bi`) and `import gfx`
picks the target's. What it cannot translate is left out with a note on
stderr. See [c-interop.md](c-interop.md).

## bic ts

```sh
bic ts api.bi                 # TypeScript declarations, to stdout
bic ts -o web/src/api.d.ts api.bi
```

Type-checks a program and writes TypeScript declarations for the
structs, enums and type aliases marked `@ts`, in the shape
`std.encoding.json`'s typed encoder writes them (`json.encode`, with the
`@serde(...)` attributes; see [stdlib.md](stdlib.md#json)), so a web
client's types match what the server sends, as ts-rs does for Rust:

```bismuth
// A host on the mesh.
@ts
@serde(renameAll: "camelCase")
struct Host {
    HostId host_id;
    str? address;
    @serde(skipIfNull) str? os;
    Hp health;
}

@ts type HostId = str;
@ts type Hp = int in 0..=100;

@ts
@serde(tag: "type")
enum Event { Joined(str host, int at), Ping }
```

```ts
// Code generated by bic ts from api.bi. DO NOT EDIT.

/** A host on the mesh. */
export interface Host {
    hostId: HostId;
    address: string | null;
    os?: string;
    health: Hp;
}

export type HostId = string;

/**
 * @minimum 0
 * @maximum 100
 */
export type Hp = number;

export type Event =
    | { type: "Joined"; host: string; at: number }
    | { type: "Ping" };
```

| Bismuth | TypeScript |
|---|---|
| integers, floats, ranged and distinct numbers | `number` (a ranged type marked `@ts` gets `@minimum` and `@maximum`) |
| `str`, `char` | `string` |
| `bool` | `boolean` |
| `T?` | `T \| null`; a field `@serde(skipIfNull)` or `skipIfEmpty` leaves out is optional, `name?: T` |
| `List<T>` | `T[]` |
| `[n]T`, vectors, `quat`, `mat4` | a tuple of n (`T[]` past 16) |
| `(A, B)` | `[A, B]` |
| `Map<str, V>` | `Record<string, V>` |
| struct | `interface` of the fields the encoder writes, renamed as `@serde` says (`priv` and `skip` fields left out); flattened structs are `& Other`, a flattened map an index signature |
| generic struct or enum | declared once over its type parameters, `Page<T>`, and used as `Page<User>` |
| enum without data | `"A" \| "B"` |
| enum with data | a union tagged as the encoder tags it: `{ Name: data }` and `"Name"`; `{ type: "Name"; ...fields }`; `{ t: "Name"; c: data }`; or untagged, the data alone (`null` for a variant without) |
| `int ! E`, `time.Weekday` and other std enums | written out in place |
| `time.Date` and other std structs of public fields only | declared, after the program's own types |
| a type with its own `serialize<W>` method (`json.Value`, `json.RawValue`) | `unknown`, or what `@ts(as: "...")` on it says |
| pointers, `Shared<T>`, `time.Zone` and other std handles (std structs with private fields), function values, maps with other keys | errors, as for `json.encode` |

**What is declared.** Every type marked `@ts` in the program's own
modules, and every struct and enum they refer to, marked or not (each
needs a name). Type aliases are different: a plain, ranged or `distinct`
alias marked `@ts` is declared and named where it is used; one that is
not is written out in place (`type Tags = List<str>;` is `string[]`).
Declarations come in source order, modules by name, then the std
structs they reach, with the `//` comment above each type, field and
variant as JSDoc. Two types with one
TypeScript name are an error, as is a program with no `@ts` type.

| Attribute | On | Does |
|---|---|---|
| `@ts` | struct, enum, `type` | declare it |
| `@ts(name: "Name")` | struct, enum, `type` | under another name |
| `@ts(as: "string")` | struct, enum, `type`, field, a variant's value | this TypeScript type instead (for a type written by its own `serialize<W>`) |
| `@ts(optional)` | field, a variant's named value | `name?: T` (a `T?` the client may leave out when it sends one; decoding takes it as `null`) |
| `@ts(skip)` | variant | left out of the enum's declaration (the program still writes it) |

`tests/ts/` has programs with their expected declarations; the test
runner checks the declarations, and with `tsc` in PATH type-checks them
and the values each program encodes against them.

## bic fmt

```sh
bic fmt file.bi             # print the formatted file
bic fmt -w src/             # rewrite every .bi file under src/
bic fmt --check .           # list unformatted files, exit 1 if any (for CI)
bic fmt < in.bi > out.bi    # stdin to stdout
bic fmt --fix -w .          # also rewrite old syntax: bodies after ->, Rc<T>
```

`--fix` first rewrites what the language no longer accepts, then formats.
Today that is the old name `Rc<T>`, which is `Shared<T>` (a file that
declares a type of its own named `Rc` is left alone), and the old
function form: `fn f() : T -> { ... }` becomes
`fn f() : T { ... }`, `-> unsafe { ... };` becomes `unsafe { ... }`, and an
expression body `-> expr;` becomes `{ expr }` (with the braces on lines of
their own when it spans lines). It works from the parser, so strings,
comments and nested braces are safe; a file with other syntax errors is
left alone.

The formatter normalizes layout and never changes a token:

- Lines are indented by bracket nesting, one level per line that opens
  brackets (`foo(a, {` indents its body once, not twice). A line that starts
  with closing brackets lines up with the line that opened them.
- A line that continues an expression gets one more level: it starts with
  `.`, `?.` or a binary operator, or the line before ended with one, or with
  `=`, `=>` or `:`.
- Trailing whitespace is removed, runs of blank lines collapse to one, and
  the file ends with one newline. CRLF files stay CRLF.
- Spacing inside a line is kept as written, so aligned struct fields and
  match arms stay aligned. The inside of multi-line strings and block
  comments is copied unchanged.

`--tabs` indents with tabs, `--indent n` sets the width (default 4). It
refuses files with unbalanced brackets or unterminated strings or comments,
naming the line.

## bils, the language server

`bils` speaks LSP 3.17 over stdio (`bic lsp` is the same server). It runs the
compiler's own lexer, parser and checker on the editor's unsaved text, so
editor diagnostics are exactly what `bic check` reports.

| Feature | Notes |
|---|---|
| Diagnostics | errors and naming-lint warnings as you type (250 ms after the last edit). Errors in an imported module are also shown on the `import` line of the file that imports it. |
| Hover | the declaration as source (`fn takeHit(ref self, int dmg)`, whole short structs and enums), its `//` doc comment, the inferred type of `let`/`var` bindings and narrowed types (`Player?` → `Player` inside `if me != null`), and the module a declaration comes from (`from the standard library, std.window`, `from module game.world`). A binding's, parameter's, field's or constant's hover ends with its type, each declared type in it a link to its definition (`type: [Event](file:///…/window/window.bi#L55,6)`, the form VS Code opens), so an editor can show that type's hover in turn. A project file the editor hasn't opened is answered too, read from disk. |
| Go to definition | locals, parameters, fields, methods, variants, types, constants, functions in other modules, and the standard library: std sources are written to the user cache directory (`~/.cache/bismuth/std-<hash>/`) so they open as files. An editor that opens only its project's files can ask for one with `bismuth/source` (`{ textDocument: { uri } }` → `{ source, path }`, null outside the std folder; advertised as `experimental.bismuthSource`). A std file is answered by its uri in that folder whether or not it was opened: semantic tokens, hover, document symbols and definitions treat it as part of its std module (a `_windows.bi` file checked for Windows). |
| Go to type definition | the struct or enum of a value's type, inferred or written. |
| Code actions | quick fixes (`quickfix`) for the problems on the lines asked about: `import std.io.fs;` for an undefined std module, `ref` at a call that passes a `ref` parameter or mutates its receiver, `var` for a `let` binding that changes, `i32(x)` where a conversion is asked for, `?` (in a function returning a result) or `.expect(...)` for a result left unhandled, renaming a declaration to the case the naming lint asks for, and the old `->` function bodies rewritten with braces. |
| References, highlights | every use of the declaration in the analyzed program. |
| Rename | locals, parameters, fields, functions, methods, types, variants and constants declared in the workspace; std and C symbols are refused. |
| Completion | after `.`/`?.`: fields, methods (only those whose `impl` applies to the receiver), variants and static functions of a type, public members of a module or C header; elsewhere: locals in scope, module declarations, the prelude, primitive types, std modules, keywords and snippets; after `import`: std modules and sibling `.bi` files. |
| Signature help | parameters of the function being called, with the active one; `self` is left out for method calls. |
| Formatting | `bic fmt` with the editor's tab size and tabs/spaces setting. |
| Inlay hints | the inferred type before each `let`/`var` name written without one. |
| Semantic tokens | types, structs, enums, functions, methods, parameters, variables, fields, variants, modules; modifiers `declaration`, `readonly` (`let`, constants) and `defaultLibrary` (std). |
| Symbols | document outline (structs with fields, enums with variants, impls with methods, functions, constants, types) and workspace symbol search over every `.bi` file under the workspace folders. |

Positions use UTF-8 when the client offers it and UTF-16 otherwise.

Settings (`initializationOptions` or the `bismuth` section of
`workspace/didChangeConfiguration`):

```json
{ "lint": true, "target": "windows" }
```

`lint: false` silences the naming-convention warnings, like `bic --no-lint`.
`target` (`linux`, `windows` or `macos`) checks programs against that OS's
standard library and C headers, like `bic --target`; without it they are
checked for the machine the server runs on, as `bic` builds by default.

Flags: `bils --log FILE` (or `--log -` for stderr) logs each analysis and
its time; `--delay 500ms` changes the diagnostics delay.

### How it works

`compiler.Analyze` parses and checks a file and everything it imports, using
open editor buffers in place of files on disk, then builds an index from
source positions to declarations. The checker already records what each
name resolves to on the AST (`Ident.Sym`, `Member.Field`, `Call.Inst`, ...),
and checks each generic function once per instantiation on a clone that
keeps the original positions, so the index walks every checked body and
keys each resolved name by its position. Generic functions that are never
instantiated are not type-checked; their globals still resolve, their
locals do not.

Completion after a dot patches the text with a placeholder name when
nothing has been typed after the dot yet, analyzes it even if the edit left
the file with syntax errors, and lists the members of the receiver's type.
The compiler keeps some package-level tables; `compiler.ResetState` clears
them before each analysis so a long-running server neither grows nor mixes
up analyses, and the server runs one analysis at a time.

### Limits

- References and rename cover the analyzed program: the file and the modules
  it imports. A module imported by several programs is renamed in the
  program you rename from.
- Named arguments (`spawn(origin, count: 40)`) are not renamed with their
  parameter.
- Hover and navigation inside std files opened from a definition are not
  supported; they show no diagnostics.

## Editors

See [`editors/`](../editors): a VS Code extension with a TextMate grammar,
Neovim and Helix configurations. Sulfur ships Bismuth as an optional
language extension.
