# bimake

bimake builds Bismuth projects the way CMake builds C and C++ ones. A
project's root holds a `BiMakeLists.txt`, written in CMake's own language,
that names its executables and libraries, their import folders and what
they link; bimake configures a build folder from it and builds every
target with bic, each after the libraries it needs.

```cmake
bimake_minimum_required(VERSION 1.0)
project(arena VERSION 1.2.0)

add_subdirectory(engine)            # engine/BiMakeLists.txt: add_library(engine ...)

add_executable(arena src/main.bi)
target_link_libraries(arena PRIVATE engine)
target_compile_options(arena PRIVATE --allocator bismuth)

enable_testing()
add_test(NAME smoke COMMAND arena --selftest)

install(TARGETS arena engine)
```

```bash
bimake -S . -B build                 # configure: writes build/BiMakeCache.txt
bimake --build build                 # build every target (the default, all)
bimake --build build --config Release --target arena -j
bimake --test build                  # build, then run the add_test() tests
bimake --install build --prefix ~/.local
bimake                               # in the project folder: configure ./build once and build it
```

bimake is installed beside bic and bils (`bic install`, the release
packages), and `bic make ...` is the same command. A CMakeLists.txt habit
carries over: `cmake_minimum_required` is `bimake_minimum_required`, and
every `CMAKE_*` variable reads as its `BIMAKE_*` name (`CMAKE_BUILD_TYPE`,
`CMAKE_CURRENT_SOURCE_DIR`, `CMAKE_SYSTEM_NAME`).

## How a Bismuth target builds

bic compiles a program from one file: the main file, and the modules it
imports, found beside it and in the `-I` folders. So a target's **first
source is the file bic builds**; the other sources listed are its modules
(bic finds them through their imports, and bimake rebuilds when they
change).

| Command | bimake runs | Writes |
|---|---|---|
| `add_executable(app main.bi)` | `bic build ... -o build/app main.bi` | `app` (`app.exe` for Windows) |
| `add_library(lib SHARED lib.bi)` (or `MODULE`) | `bic build --library ...` | `liblib.so` (Linux), `lib.dll` (Windows) |
| `add_library(lib STATIC lib.bi)` (or `OBJECT`, or no kind) | `bic check ...` | nothing: a module library is Bismuth source its users import, checked on its own |
| `add_library(lib INTERFACE)` | nothing | import folders and flags for its users |

**Import folders.** A library's own folder (its `BiMakeLists.txt`'s) is
an import folder of every target that links it, so an executable that
links `engine` can `import physics;` for `engine/physics.bi`.
`target_include_directories(t PUBLIC|PRIVATE|INTERFACE dirs...)` adds more
(PUBLIC: for `t` and its users, PRIVATE: for `t` alone, INTERFACE: its
users alone), and `include_directories(dirs...)` adds them to the targets
made after it in that folder and the ones below. They become bic's `-I`
flags; the main file's folder is always bic's first.

**Links.** `target_link_libraries(t [PUBLIC|PRIVATE|INTERFACE] items...)`
takes targets, whose import folders and INTERFACE options `t` gets (and,
through PUBLIC and INTERFACE links, theirs), and `.a`, `.o` or `.lib`
files, which go to bic as `--link-static` for the program's `extern c fn`s
(docs/c-interop.md). A shared library target is built first and copied
beside each executable that links it, where the executable loads it from
(Linux programs look beside themselves first, RUNPATH `$ORIGIN`). The
program declares the library's functions with `@link("lib")
extern c fn ...`, in a module it imports, usually in the library's folder.

**Flags.** `target_compile_options(t ...)` and `add_compile_options(...)`
are bic flags: `-O3`, `--no-lint`, `--unchecked`, `--allocator bismuth`,
`--icon app.png`, `--app-name ...` and the rest of `bic build --help`.
`WIN32` in `add_executable` (or the `WIN32_EXECUTABLE` property) is bic's
`--gui` on Windows. The configuration picks the profile:

| `--config` / `BIMAKE_BUILD_TYPE` | bic flags |
|---|---|
| `Debug` (the default) | `--debug` (`-Og -g`) |
| `Release` | `--release` (`-O2`) |
| `RelWithDebInfo` | `--release -g` |
| `MinSizeRel` | `-Os` |

**Other systems.** `-D BIMAKE_SYSTEM_NAME=Windows` (or `Linux`, `Darwin`)
and `-D BIMAKE_SYSTEM_PROCESSOR=arm64` build for another system with bic's
`--target` and `--arch`; `WIN32`, `UNIX`, `APPLE` and `LINUX` follow the
system built for. bic writes shared libraries for Linux and Windows, not
macOS yet.

**Up to date.** A target is rebuilt when its bic command changed, when a
`.bi` file in its main file's folder or its import folders (and the
folders under them) is newer than its output, when a BiMakeLists.txt or
bic changed, or when a library it links was rebuilt. Each build folder's
`.bimake/` keeps the commands.

## The build folder

Configuring writes `BiMakeCache.txt`, CMake's `CMakeCache.txt`:
`NAME:TYPE=VALUE` lines holding the `option()`s, `set(... CACHE ...)`
entries and `-D` values. Edit a value there, or give `-D NAME=VALUE`, and
configure again (`--fresh` starts the cache over). Building reads the
project's BiMakeLists.txt again each time, with that cache, so a changed
list takes effect without configuring.

`bimake_targets.json` beside it lists each target's type, main file,
output, import folders and bic command, for editors and other tools (as
CMake's `compile_commands.json` is for C). `bimake --targets build
--json` prints the same.

Cache entries bimake sets: `BIMAKE_BUILD_TYPE`, `BIMAKE_BIC` (the bic it
runs: `$BIMAKE_BIC`, else the bic beside bimake, else bic on PATH),
`BIMAKE_INSTALL_PREFIX` (`/usr/local`, or `C:/Program Files/<project>` on
Windows) and `BIMAKE_HOME_DIRECTORY` (the source folder).
`BIMAKE_BIC_FLAGS` is a list of bic flags for every target.

## The command line

```
bimake [-S src] [-B build] [-D NAME[:TYPE]=VALUE]... [--fresh] [path]
bimake --build build [--target t]... [--config c] [-j [n]] [--clean-first] [-v] [-- bic flags]
bimake --install build [--prefix dir] [--config c]
bimake --test build [-R text] [-V]
bimake --targets build-or-source [--json]
```

- Configure: `-S` the source folder (default: the one the build folder
  was configured for, else the current one), `-B` the build folder
  (default: the current one). A bare path is a source folder, or a
  build folder to configure again.
- `--build`: `--target` names targets (repeatable, or comma-separated;
  `all` is the default, `clean` removes what was built), `-j n` builds n
  targets at once (`-j` alone: one per CPU), `-v` prints each bic command,
  and what follows `--` goes to every bic. A target that fails stops the
  ones that need it; bic's errors print as bic prints them,
  `file:line:col: error: ...`, as do bimake's own for a BiMakeLists.txt.
- `--test` builds, then runs each test (`-R` keeps those whose names
  contain the text, `-V` shows their output as they run) and exits 8 if
  one fails, as ctest does.
- `--install` copies what the `install()` commands name under the prefix.
- `--quiet` leaves out the progress lines.

## Commands

The BiMakeLists.txt language is CMake's: `command(arg ...)`, `# comments`
and `#[[ bracket comments ]]`, `"quoted"` arguments, `[[bracket]]`
arguments taken as written, `${VAR}` and `$ENV{VAR}` references, and lists
as `a;b;c` (an unquoted argument splits into its elements). Command names
ignore case. bimake has:

| | |
|---|---|
| Project | `bimake_minimum_required(VERSION v)` (`cmake_minimum_required`), `project(name [VERSION v] [DESCRIPTION d])`, `cmake_policy`, `enable_testing` (accepted and ignored) |
| Variables | `set(VAR values... [PARENT_SCOPE])`, `set(VAR value CACHE TYPE doc [FORCE])`, `unset`, `option(NAME doc [ON\|OFF])`, `list(APPEND\|PREPEND\|LENGTH\|GET\|FIND\|JOIN\|REMOVE_ITEM\|REMOVE_DUPLICATES\|SORT\|REVERSE ...)`, `string(TOUPPER\|TOLOWER\|LENGTH\|STRIP\|APPEND\|CONCAT\|REPLACE\|FIND\|SUBSTRING\|COMPARE ...)`, `math(EXPR var expr)` |
| Files | `file(GLOB\|GLOB_RECURSE var [RELATIVE dir] patterns...)`, `file(READ\|WRITE\|APPEND\|MAKE_DIRECTORY ...)`, `get_filename_component(var file NAME\|DIRECTORY\|NAME_WE\|EXT\|ABSOLUTE)`, `cmake_path(GET var FILENAME\|STEM\|EXTENSION\|PARENT_PATH out)`, `include(file [OPTIONAL])`, `find_program(VAR names... [REQUIRED])` |
| Flow | `if()/elseif()/else()/endif()`, `foreach(v items...)`, `foreach(v RANGE [lo] hi [step])`, `foreach(v IN LISTS l... ITEMS i...)`, `while()/endwhile()`, `break()`, `continue()`, `return()`, `function()/endfunction()` (with `ARGC`, `ARGV`, `ARGVn`, `ARGN`), `macro()/endmacro()` (its arguments are variables of the caller's scope) |
| Folders | `add_subdirectory(dir [bindir] [EXCLUDE_FROM_ALL])`: a scope of its own, as in CMake |
| Targets | `add_executable(name [WIN32] [EXCLUDE_FROM_ALL] sources...)`, `add_library(name [STATIC\|SHARED\|MODULE\|INTERFACE\|OBJECT] [EXCLUDE_FROM_ALL] sources...)` (`BUILD_SHARED_LIBS` makes a library with no kind shared), `target_sources`, `target_include_directories`, `target_link_libraries`, `target_compile_options` (`target_link_options`), `include_directories`, `add_compile_options` (`add_link_options`), `link_libraries`, `add_dependencies` |
| Properties | `set_target_properties(targets... PROPERTIES name value ...)`, `set_property(TARGET targets... [APPEND] PROPERTY name values...)`, `get_target_property(var target name)`: `OUTPUT_NAME`, `PREFIX`, `RUNTIME_OUTPUT_DIRECTORY`, `LIBRARY_OUTPUT_DIRECTORY`, `WIN32_EXECUTABLE`, `EXCLUDE_FROM_ALL`, `INCLUDE_DIRECTORIES`, `INTERFACE_INCLUDE_DIRECTORIES`, `COMPILE_OPTIONS`, `LINK_LIBRARIES`, and to read `TYPE`, `SOURCES`, `SOURCE_DIR`, `BINARY_DIR`; any other name is kept for `get_target_property` |
| Tests | `add_test(NAME name COMMAND command args... [WORKING_DIRECTORY dir])` (a command that names an executable target runs its output) |
| Install | `install(TARGETS targets... [RUNTIME DESTINATION d] [LIBRARY DESTINATION d])` (default `bin`, and `lib` for Linux shared libraries), `install(FILES\|PROGRAMS files... DESTINATION d)`, `install(DIRECTORY dirs... DESTINATION d)` (`dir/` copies what is in it) |
| Messages | `message([STATUS\|WARNING\|SEND_ERROR\|FATAL_ERROR\|...] text...)` |

`if()` has CMake's conditions: constants (`1 ON YES TRUE Y`, `0 OFF NO
FALSE N IGNORE NOTFOUND`, `*-NOTFOUND`, the empty string), a variable's
name for its value, `NOT`, `AND`, `OR`, parentheses, `DEFINED`, `EXISTS`,
`IS_DIRECTORY`, `IS_ABSOLUTE`, `TARGET`, `COMMAND`, `EQUAL LESS GREATER
LESS_EQUAL GREATER_EQUAL`, `STREQUAL STRLESS STRGREATER STRLESS_EQUAL
STRGREATER_EQUAL`, `VERSION_EQUAL VERSION_LESS VERSION_GREATER
VERSION_LESS_EQUAL VERSION_GREATER_EQUAL`, `MATCHES` (std.text.regex) and
`IN_LIST`.

Variables bimake sets: `BIMAKE_SOURCE_DIR`, `BIMAKE_BINARY_DIR`,
`BIMAKE_CURRENT_SOURCE_DIR`, `BIMAKE_CURRENT_BINARY_DIR`,
`BIMAKE_CURRENT_LIST_FILE`, `BIMAKE_CURRENT_LIST_DIR`, `BIMAKE_VERSION`,
`BIMAKE_SYSTEM_NAME`, `BIMAKE_HOST_SYSTEM_NAME`, `WIN32`, `UNIX`,
`APPLE`, `LINUX`, `PROJECT_NAME`, `PROJECT_VERSION` (and `_MAJOR`,
`_MINOR`, `_PATCH`, `_TWEAK`), `PROJECT_SOURCE_DIR`, `PROJECT_BINARY_DIR`,
`<name>_SOURCE_DIR`, `<name>_BINARY_DIR`, `BIMAKE_PROJECT_NAME`.

Not in bimake: generators (there is one way to build), `add_custom_command`
and `add_custom_target`, `execute_process`, `configure_file`,
`find_package`, imported and alias targets, generator expressions
(`$<...>`), and compile definitions (Bismuth has no preprocessor; use an
option and `if()` to pick sources or flags). An unknown command is an
error, so a list that uses one says so at configure time.

## Editors

The language server (bils) checks a file of a bimake project with the
project's import folders: for the target whose sources hold the file, or
whose main file's folder holds it, its main file's folder and its `-I`
folders, read from the project's top BiMakeLists.txt (the nearest one
above the file, then up while the parent folder has one too) with the
options of its `build/` folder's cache. So an import that builds also
resolves in the editor. Sulfur builds a folder with a BiMakeLists.txt with
bimake, its targets and configurations in its build menu.
