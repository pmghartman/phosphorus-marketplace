# Windows, rendering and GUI

Status: design, plus three working slices. The first (issue #10) has
`std window`, `std gfx` (clear and present), `std dl`, `std objc`, C function
pointer types and `bic build --gui`. Its example is `examples/window`.
Native Wayland (#18) followed: `std window` on `xdg-shell` with
libdecor or server-side decorations, xkbcommon text, and a Vulkan
surface for it in `std gfx`. The second (#16) makes `gfx` draw: buffers,
images, samplers, shaders, pipelines and bindings behind generational
handles, per-thread encoders, Vulkan dynamic rendering, Direct3D 11 and
Metal to match, a new Direct3D 12 backend, headless devices, the
`examples/triangle` and `examples/sprites` programs, and the benchmarks
against bgfx in `bench/gfx` (§4). The third (#63) adds compute passes and
storage buffers, mipmaps, MSAA targets and resolves, sampling depth
images, sorted passes, translating a pass's encoders in parallel, GPU
timestamps, and a pipeline cache on disk for Metal (§2, "Slice 3").

Decisions taken with the owner:

| Question | Decision |
|---|---|
| Renderer API shape | sokol_gfx / WebGPU style: explicit immutable objects, passes, per-thread encoders |
| Shaders | Written in Bismuth and compiled by bic to SPIR-V, MSL and HLSL (done, #17: [shaders.md](shaders.md)) |
| Calling Vulkan, COM and Objective-C | A C function pointer type (`extern c fn(...) : R`), with libraries loaded at run time |
| Windows backends | Vulkan, Direct3D 12 and Direct3D 11 (D3D12 and D3D11 fall back to WARP) |

## 1. Platform layer

### What others do

| | SDL3 | GLFW | sokol_app | winit |
|---|---|---|---|---|
| Event model | poll a queue (`SDL_PollEvent`) or callbacks (main callbacks) | callbacks per kind, `glfwPollEvents` pumps | the app is callbacks (`init/frame/event/cleanup`); sokol owns the loop | an `EventLoop` closure (or the newer `ApplicationHandler` trait) owns the loop |
| Who owns `main` | you | you | sokol (`sokol_main`) | winit, on macOS and the web |
| Windows | many | many | one | many |
| Keys | scancodes and keycodes, text separately | keys (US-layout names of physical keys), text via char callback | keycodes and chars | physical `KeyCode` and logical `Key` |
| HiDPI | pixel and point sizes both reported | framebuffer vs window size | `sapp_dpi_scale` | `scale_factor`, physical vs logical sizes |
| Linux | X11 and Wayland, chosen at run time | X11 and Wayland, chosen at build time (run time since 3.4) | X11 only (GLX/EGL) | X11 and Wayland at run time |
| Graphics | GPU API, renderer, GL contexts | GL/Vulkan surfaces only | sokol_gfx glue | none (raw handles via `raw-window-handle`) |

What Bismuth takes from them:

- **SDL3's polling queue.** `for e in w` pumps the OS queue and never blocks.
  The game keeps its loop and `main`, and events are plain enum values that
  `match` handles exhaustively. That is a better fit for Bismuth than a
  callback table.
- **GLFW's split between window and framebuffer sizes**, stated once:
  `Resize` and `width()`/`height()` are framebuffer pixels.
- **winit's raw handles.** `w.native()` gives the OS objects
  (`wl_display*` and `wl_surface*`, `xcb_connection_t*` and window id,
  `HINSTANCE` and `HWND`, `NSWindow` and `NSView`), so `gfx` or any C
  library can draw into the window.
- **Physical keys** (SDL scancodes, winit `KeyCode`). The three OS halves
  convert their key numbers to Linux evdev codes and map those to one `Key`
  enum. Text (UTF-8, from the keymap and the input method) is a separate
  event kind, `Text`, sent only after `w.textInput(true)`: games bind
  physical keys and get no input method in their way, and text is a GUI
  concern. An input method's composition, the text not committed yet, is
  `Preedit(text, caret)` (`""` when it ends), for the program to show at
  its caret, whose box `w.setTextInputRect(x, y, w, h)` gives the input
  method for its candidate list. Wayland: xkbcommon and
  `zwp_text_input_v3` (below). X11: the core keyboard mapping (Shift, Caps
  Lock, AltGr as Mod5, Num Lock), keysyms to UTF-8 through libxkbcommon's
  `xkb_keysym_to_utf32` when it is there, dead keys and `Multi_key`
  sequences through libxkbcommon's compose table for the locale (the
  sequence so far shows as `Preedit`; Wayland's keymap text does the
  same), and an input method over IBus's D-Bus protocol (no Xlib, so no
  XIM; Fcitx 5 speaks it too): when ibus-daemon serves the display
  (`IBUS_ADDRESS`, or its address file) or `org.freedesktop.portal.IBus`
  has an owner, each key goes to the input context first
  (`ProcessKeyEvent`; a key it takes is no `KeyDown`), its
  `UpdatePreeditText`/`HidePreeditText` are `Preedit`, `CommitText` is
  `Text`, and the text input rectangle is `SetCursorLocation`
  (`BISMUTH_IM=none` turns it off). Win32: `WM_CHAR` (surrogate pairs joined);
  the window has an IME context only while text input is on
  (`ImmAssociateContextEx`), and then draws the composition itself:
  `WM_IME_COMPOSITION`'s `GCS_COMPSTR`/`GCS_CURSORPOS` are `Preedit`,
  `GCS_RESULTSTR` is `Text`, the system composition window is off, the
  candidate window goes below the text input rectangle, and keys the IME
  takes (`VK_PROCESSKEY`) are no `KeyDown`. Cocoa: the content view is an
  `NSView` subclass adopting `NSTextInputClient`; with text input on, key
  presses go through `interpretKeyEvents:`, `insertText:` is `Text`,
  `setMarkedText:` `Preedit`, `firstRectForCharacterRange:` the text input
  rectangle. `tests/gui/text_x11.bi` (XTest under Xvfb) and
  `tests/gui/text_windows.bi` check the X11 and Win32 halves, and
  `tests/gui/im_x11.bi` X11's compose sequences and, with an ibus-daemon
  (`tools/ci/with-ibus.sh`), a composition through IBus.

The slice's API (`std/window/window.bi`, `import std.window;`):

```bismuth
let w = window.open({ title: "game", width: 1280, height: 720, resizable: true })?;
for e in w {
    match e {
        Close => ..., Resize(width, height) => ..., KeyDown(key, repeat, scancode) => ..., KeyUp(key, scancode) => ...,
        MouseMove(x, y) => ..., MouseDown(button) => ..., MouseUp(button) => ...,
        Scroll(dx, dy) => ..., Focus(focused) => ..., Text(text) => ..., Scale(scale) => ...,
        _ => ...,
    }
}
w.scale(); w.isFocused();                          // the content scale (DPI / 96), keyboard focus
w.minimize(); w.maximize(); w.restore(); w.state(); // Normal, Minimized, Maximized, Fullscreen; Event.State
w.setFullscreen(window.Fullscreen.Borderless);     // Off, Borderless, Exclusive(mode): bool
let ms = w.monitors(); let m = w.monitor();        // name, rectangle, work area, scale, primary, mode, modes
w.position(); w.setPosition(x, y); w.center(); w.setSize(width, height);
w.setPointerLock(true);                            // mouse look: MouseDelta(dx, dy) instead of MouseMove
w.warpPointer(x, y);                               // bool: Wayland only while locked
window.open({ title: "game", width: 1280, height: 720, center: true, fullscreen: .Borderless })?;
w.setTitle("..."); w.textInput(true); w.native(); w.close();
w.setCursor(window.Cursor.IBeam);                  // Arrow, IBeam, Hand, Crosshair, Resize*, NotAllowed, Wait, Hidden
w.setClipboard("text"); let s = w.clipboard();     // str?: null when it holds no text
w.setAccessTree(tree);                             // what screen readers read; requests come back as Access events
window.open({ title: "editor", width: 1600, height: 900, maximized: true, clientTitleBar: true })?;
w.setTitleBar(bar); w.hasClientTitleBar(); w.titleBarHit(x, y); w.requestClose();   // a client-drawn title bar
```

**Accessibility** (AccessKit-style, `access.bi`): the program describes
what the window shows as a `window.AccessTree` of `AccessNode`s (id,
`AccessRole`, name, value, bounds in window pixels, children, actions,
focusable, checked, expanded, selected, an `AccessRange` for sliders and
progress bars, a text field's caret) and gives it to the window with
`w.setAccessTree(t)` whenever it changes (std.ui does each frame through a
bound window). Screen readers' requests come back as
`Event.Access(node, AccessAction, value)`: `Press`, `Focus`, `SetValue`
(with the text), `Increment`, `Decrement`, `Expand`, `Collapse`. Node ids
are the program's and must name the same item from tree to tree.
On Linux (X11 and Wayland) the window is an AT-SPI application: when
`org.a11y.Status.IsEnabled` is on (a screen reader turns it on; the
session bus is watched for it) it joins the accessibility bus
(`org.a11y.Bus.GetAddress`, or `AT_SPI_BUS_ADDRESS`), is embedded by the
registry, and its pump answers `org.a11y.atspi.Accessible`, `Component`,
`Action`, `Value`, `Text` and `EditableText` calls from the last tree,
sending `Event.Object` signals for focus, state, value, name, text and
children changes (std.os.dbus, no libatspi). On Windows, `WM_GETOBJECT`
returns a UI Automation provider: an element per node with
`IRawElementProviderSimple`, `Fragment` and `FragmentRoot` and the
`Invoke`, `Value`, `RangeValue`, `Toggle` and `ExpandCollapse` patterns,
read from a locked copy of the tree since UI Automation calls from its
own threads, with focus and property-changed events raised while a client
listens. On macOS the content view answers `NSAccessibility` with an
`NSAccessibilityElement` per node (type-checked; not run on a Mac).
`tests/gui/access_linux.bi` reads and drives a std.ui window over D-Bus as
a screen reader would (under Xvfb and `dbus-run-session`);
`tests/gui/access_windows.ps1` does it through `System.Windows.Automation`.

**Scan codes.** `KeyDown` and `KeyUp` carry the key's scancode next to
its `Key`: the evdev code every OS half converts to (X11 keycode - 8,
`wl_keyboard.key`, the Win32 set-1 scancode with its extended bit mapped,
the macOS virtual key code through a table), so the same number on every
OS. For the main block it is the PC set-1 scancode (`0x29` is the key left
of 1); keys `Key` has no name for (the ISO key left of Z, 86; media and
language keys) are `Key.Unknown` with their own scancode, which a game can
bind and save. 0 when the OS gave none.

**Content scale** (`w.scale()`, framebuffer pixels per 1/96-inch logical
pixel, and `Event.Scale(scale)` on each change) sizes a game's text and UI.
X11: `Xft.dpi / 96` from the root window's `RESOURCE_MANAGER` (what `xrdb`
and the desktops' settings daemons set; one scale for the whole screen),
read again on its `PropertyNotify`; 1 without one. Wayland: the surface's
`preferred_buffer_scale` (wl_compositor 6), else the largest scale of the
`wl_output`s it entered; the window sets that buffer scale on its surface,
so sizes stay framebuffer pixels (the logical size times the scale) and
`Options`' size is logical, as points are on macOS. Integer scales only
(no `wp_fractional_scale_v1` yet). Win32: `GetDpiForWindow / 96` (the
window is per-monitor DPI aware); `WM_DPICHANGED` takes the suggested
rectangle and reports the new scale. Cocoa: `backingScaleFactor`, again on
`windowDidChangeBackingProperties:`. `tests/gui/scale_x11.bi` (under Xvfb)
and `tests/gui/scale_wayland.bi` (under `weston --scale=2`) check it.

**Window state, fullscreen and monitors.** `w.state()` and
`Event.State(WindowState)` say whether the window is `Normal`,
`Minimized` (a game stops drawing), `Maximized` or `Fullscreen`;
`minimize()`, `maximize()` and `restore()` ask for them (restore undoes a
minimize, else a maximize). `setFullscreen(Borderless)` covers the
monitor the window is on at its mode, `Exclusive(mode)` switches that
monitor to one of its `modes` first, and `Off` puts back the position,
size and maximized state the window had; it returns false when the
monitor has no such mode or the window manager can't. `monitors()` (the
primary first) and `monitor()` (the one with most of the window) give a
`Monitor`: name, rectangle and work area in desktop coordinates (pixels on
X11 and Win32, points on macOS, the compositor's logical space on
Wayland), scale, `primary`, the current `VideoMode` (pixels and hertz) and
all its modes, largest first. `position()`, `setPosition(x, y)`,
`center()` (in the monitor's work area) and `setSize(w, h)` place the
drawable area; `Options` takes `x`, `y`, `center` and `fullscreen` too.

- X11: with an EWMH window manager (`_NET_SUPPORTING_WM_CHECK`), the
  window sends it `_NET_WM_STATE` client messages (maximized, fullscreen)
  and `WM_CHANGE_STATE` (minimize), maps itself to restore, and reads its
  state from `_NET_WM_STATE` and `WM_STATE` as they change; positions use
  static gravity, so they are the drawable area's. With none (a bare
  server, Xvfb) the window does each itself: unmaps, covers the work area
  or the monitor, and goes back. Monitors are RandR's connected outputs
  (`libxcb-randr`, RandR 1.3: `GetScreenResourcesCurrent`, the primary
  output, refresh from the dot clock and totals), the work area
  `_NET_WORKAREA`'s part; without RandR, the screen. Exclusive
  fullscreen sets the output's CRTC to the mode (`RRSetCrtcConfig`) and
  back when the window leaves fullscreen or closes.
- Wayland: `xdg_toplevel` `set_maximized`, `set_minimized` and
  `set_fullscreen` on the output the window is on (libdecor's
  `libdecor_frame_*` when it decorates the window), the state from each
  configure (`maximized`, `fullscreen`, and `suspended`, xdg-shell 6, as
  minimized), the windowed size back from before when a configure leaves
  it to the client. **What a compositor refuses:** a client can't place
  its window or learn where it is (`position()` is null, `setPosition`
  and `center` return false and `Options.x`, `y` and `center` do
  nothing), can't un-minimize itself (`restore` only un-maximizes), is
  not told it is minimized unless the compositor sends `suspended`, and
  can't switch an output's mode (`Exclusive` is borderless). Monitors are
  the `wl_output`s with their geometry, modes (most compositors announce
  only the current one), scale and name.
- Win32: `ShowWindow` and `WM_SIZE` (a minimized window sends no
  `Resize`); fullscreen keeps the style and `WINDOWPLACEMENT`, sets
  `WS_POPUP` and covers `MonitorFromWindow`'s monitor, then puts both
  back (maximizing again after, so the frame fits). Exclusive fullscreen
  is `ChangeDisplaySettingsExW` with `CDS_FULLSCREEN`; the window
  minimizes when it loses the focus and the desktop's mode comes back
  while it is minimized. Monitors are `EnumDisplayMonitors` with
  `GetMonitorInfoW`, names from `EnumDisplayDevicesW`, modes from
  `EnumDisplaySettingsW` (32 bits a pixel), scale from `GetDpiForMonitor`.
- Cocoa: `miniaturize:`, `zoom:`, `deminiaturize:`, `toggleFullScreen:`
  (the window gets its own space), the delegate reporting each change;
  exclusive fullscreen first `CGDisplaySetDisplayMode`, undone with
  `CGRestorePermanentDisplayConfiguration`. Monitors are `NSScreen`s
  (frame, `visibleFrame`, `backingScaleFactor`, `localizedName`) with
  CoreGraphics' modes. Type-checked, not run on a Mac.

`tests/gui/display_x11.bi` (Xvfb, no window manager), `wm_x11.bi` (Xvfb
with a small EWMH window manager in the test), `display_wayland.bi`
(headless weston, with and without libdecor) and `display_windows.bi`
check it. `Options.maximized` opens the window maximized (on Windows with
its restored size fitted to the monitor's work area and put at its
corner, unless `x`, `y` or `center` place it).

**Client-drawn title bar** (`titlebar_*.bi`). With
`Options.clientTitleBar` the program draws the title bar itself in the
top of the drawable area (its icon, menus, title and window buttons, as
editors and browsers do) and says every frame where its parts are:

```bismuth
let w = window.open({ title: "editor", width: 1600, height: 900, clientTitleBar: true, maximized: true })?;
// each frame, after laying the bar out (rectangles in pixels, like mouse positions)
w.setTitleBar(window.TitleBar {
    caption: window.HitRect { x: 0, y: 0, width: w.width(), height: 32 },   // drags the window
    minimize: minRect, maximize: maxRect, close: closeRect,                 // the program's buttons
    exclude: [menusRect],                                                   // controls on the bar
});
if w.hasClientTitleBar() { /* draw the window buttons; a click: w.minimize(), w.maximize() or w.restore(), w.requestClose() */ }
w.titleBarHit(x, y);    // TitleBarHit: Client, Caption, Minimize, Maximize, Close, Top, Left, ..., BottomRight
```

The window answers the OS's hit test from the last `TitleBar`
(`window.hitTitleBar`, a plain function a test can call): a resize band
along the edges the window has no frame on, the buttons, the excluded
controls (the program's), the caption, the program's elsewhere. Presses
on the caption and the edges are the OS's (moving, snapping, resizing)
and are not reported as `MouseDown`; the buttons and controls are the
program's ordinary clicks, and the program acts on them (`requestClose()`
reports `Close` as the system's close would, so one path asks to save).
`hasClientTitleBar()` says whether the OS took it; when false the window
has the system's title bar and the program draws no window buttons.

- Win32: `WM_NCCALCSIZE` gives the caption back to the client area and
  keeps the system frame on the left, right and bottom (the resize
  border, the drop shadow and Snap; a one-pixel edge on Windows 10 and
  11); a maximized window's top is inset by the frame it hangs off the
  monitor, less a sliver for an auto-hide taskbar. `WM_NCHITTEST` asks
  the frame first, then answers `HTTOP`/`HTTOPLEFT`/`HTTOPRIGHT` in a
  band half the frame deep along the top edge, `HTMINBUTTON`,
  `HTMAXBUTTON` (where Windows 11 opens Snap Layouts) and `HTCLOSE` for
  the buttons, `HTCLIENT` for the controls and `HTCAPTION` for the bar,
  so dragging, Aero Snap, the double click and the window menu are the
  system's. The buttons' non-client mouse messages come to the program as
  `MouseMove`, `MouseDown` and `MouseUp` at client positions (with
  `WM_NCMOUSELEAVE` tracked), and their presses are not passed to
  `DefWindowProc`, which would draw the classic caption buttons; the
  frame is not extended into the client area (`DwmExtendFrameIntoClientArea`
  is not needed: the kept frame draws the shadow), and `WM_NCACTIVATE`
  repaints no caption.
- X11: `_MOTIF_WM_HINTS` asks the window manager for no decorations. A
  caption press that moves more than a few pixels lets go of its grab and
  sends `_NET_WM_MOVERESIZE` (move); two caption presses within 400 ms
  maximize or restore; a band 6 logical pixels deep along each edge sends
  `_NET_WM_MOVERESIZE` with the edge. Over a band the pointer shows that
  edge's resize cursor from the cursor theme (`n-resize` ... `se-resize`,
  else `top_side` ... `bottom_right_corner`, else the core cursor font's),
  as a window manager's frame does, whatever `setCursor` asked for, and
  the program's cursor again once it leaves. Without a window manager the
  bar cannot move the window; the double click still works.
- Wayland: no libdecor frame and no server-side decoration mode are
  asked for. The caption is `xdg_toplevel.move` once a press moves (with
  the press's serial), two presses within 400 ms maximize or restore, a
  right click is `show_window_menu`, and the edge bands are
  `xdg_toplevel.resize`, showing the edges' resize cursors as on X11.
- Cocoa: not taken (`hasClientTitleBar()` is false): the window keeps its
  own title bar and traffic-light buttons.

`w.setBorderColor(BorderColor)` (`Default`, `None`, `Rgb(r, g, b)`)
colours the one-pixel border Windows 11 draws around a window
(`DwmSetWindowAttribute(DWMWA_BORDER_COLOR)`), to match a client-drawn
bar; it returns false and does nothing before Windows 11 and on Linux and
macOS, which have no such border.

`tests/run/window_titlebar.bi` checks the hit test, `tests/gui/titlebar_windows.bi`
the Win32 half through `WM_NCHITTEST` (and the border colour),
`tests/gui/titlebar_x11.bi` the X11 one under Xvfb and
`tests/gui/titlebar_cursor_x11.bi` its edge cursors; `samples/titlebar.bi` draws such a bar with
std.ui (`--window` opens it as a real window's title bar).

**Pointer lock and warping.** `w.setPointerLock(true)` hides the pointer
and holds it in the window for mouse look: `MouseMove` stops and
`MouseDelta(dx, dy)` reports the motion in pixels; the lock lets go while
the window has no focus and comes back with it. `w.warpPointer(x, y)`
moves the pointer to a point of the window; while locked, it is where the
pointer shows when the lock ends.

- X11: an active pointer grab confined to the window with the hidden
  cursor; motion is measured from the last position and the pointer is
  warped back to the window's middle once a pump (as GLFW does without
  XInput2 raw motion, which needs libxcb-xinput). `xcb_warp_pointer`.
- Wayland: `zwp_pointer_constraints_v1.lock_pointer` (persistent) and
  `zwp_relative_pointer_v1`, whose unaccelerated motion is `MouseDelta`
  while the compositor says `locked`; the cursor is hidden. Compositors
  lock only when they choose (weston: the surface clicked last, with the
  keyboard focus); without the protocols `setPointerLock` returns false.
  Clients can't warp an unlocked pointer (`warpPointer` returns false);
  locked, it is `set_cursor_position_hint`.
- Win32: raw input (`RegisterRawInputDevices`, `WM_INPUT`: raw counts, or
  absolute positions from tablets and remote desktops as differences),
  the cursor hidden and clipped to the client area (`ClipCursor`) while
  the window has the focus; `SetCursorPos`.
- Cocoa: the cursor hidden, parked in the window's middle and detached
  from the mouse (`CGAssociateMouseAndMouseCursorPosition(false)`), mouse
  events' `deltaX`/`deltaY` as `MouseDelta`; `CGWarpMouseCursorPosition`.

`tests/gui/pointer_x11.bi` (XTest's relative motion under Xvfb),
`pointer_wayland.bi` (weston's X11 backend) and `pointer_windows.bi`
(`SendInput` in the desktop session) check it.

**Cursor shapes** come from the system: on X11 the desktop's cursor theme
through `libxcb-cursor` when it is installed, else the server's core
cursor font (`Arrow` is then the parent window's cursor, as before any
`setCursor`); on Wayland the theme `libwayland-cursor` loads, by CSS name
with the X11 name as a fallback, shown with `wl_pointer.set_cursor`; on
Win32 the `IDC_*` cursors, set on `WM_SETCURSOR` over the client area; on
Cocoa `NSCursor`'s (the diagonal resize ones are AppKit's private cursors
where it has them; `Wait` stays the arrow), set again as the pointer
moves. `Hidden` shows no pointer over the window (on macOS,
`[NSCursor hide]` until another shape). `tests/gui/cursor_x11.bi` reads
the shapes back through XFIXES under Xvfb, `tests/gui/cursor_windows.bi`
through `GetCursor`.

**Clipboard** (text only). X11: the `CLIPBOARD` selection; setting it
makes the window its owner, which answers `SelectionRequest`s (`TARGETS`,
`UTF8_STRING`, `STRING`, `TEXT`) from its pump, and reading converts it to
`UTF8_STRING` into a property of the window, waiting up to a second for
the owner (`INCR` transfers included) while queueing the window's other
events. The text goes when the window does, unless a clipboard manager
keeps it. Wayland: `wl_data_device_manager`, a `wl_data_source` set with
the last input event's serial and served from the pump, and the focused
window's `wl_data_offer` read through a pipe. Win32: `CF_UNICODETEXT`,
with CRLF line ends on the clipboard. Cocoa: the general `NSPasteboard`.
`tests/gui/clipboard_x11.bi` (under Xvfb, three processes),
`tests/gui/clipboard_wayland.bi` (two processes, in weston's X11 backend)
and `tests/gui/clipboard_windows.bi` check it.

`Window` is a handle to heap state that never moves, because the OS
callbacks (`WndProc`, Cocoa delegates) find the window by its address.

### Windows (Win32)

`RegisterClassExW`/`CreateWindowExW`, with a `PeekMessageW` loop in `next()`.
The `WndProc` is an `export c fn` that takes scalars, which the Win64 backend
already supports. The window's state pointer goes in as
`CreateWindowExW`'s parameter and is stored in `GWLP_USERDATA` at
`WM_NCCREATE`.

- **DPI.** `SetProcessDpiAwarenessContext(PER_MONITOR_AWARE_V2)` is looked
  up at run time and used when it exists. Sizes are physical pixels.
  `WM_DPICHANGED` resizes the window to the rectangle Windows suggests and
  sends `Scale`.
- **Keys** come from the scancode in `lParam` (set 1, with an extended bit).
  The scancode is layout-independent and equals the evdev code for the main
  block.
- **Resizing.** Dragging a window edge enters a modal loop inside
  `DispatchMessageW`, so frames stop while you drag. sokol_app and SDL keep
  drawing from `WM_TIMER`/`WM_PAINT` in that loop; we will do the same once
  frame callbacks exist.
- **`bic build --gui`** marks the PE image as `IMAGE_SUBSYSTEM_WINDOWS_GUI`,
  so it starts with no console. Writes to stdout and stderr then go nowhere
  unless redirected, which std's `writeAll` already tolerates.
- **Resources** (#19). Every executable has a `.rsrc` section with a
  manifest: the UTF-8 code page for all programs, and for `--gui` ones
  per-monitor-v2 DPI awareness and common controls 6. `--icon` embeds an
  icon, which the window class uses (`LoadIconW`), and GUI programs carry
  version information. See "GUI programs" in [tooling.md](tooling.md).
- **Next:** a `WM_CHAR`/`WM_IME_*` text event.

### Linux: Wayland, and X11

Both halves load their libraries with `dl` at run time, so a program that
imports `window` still starts on a machine with neither and gets
`WindowError.NoDisplay`. `window_linux.bi` picks one: **Wayland** when
`WAYLAND_DISPLAY` is set and `libwayland-client.so.0` loads and connects,
else **X11**. `BISMUTH_WINDOW=x11` or `BISMUTH_WINDOW=wayland` forces one,
as `SDL_VIDEODRIVER` does (forcing Wayland turns its failure into the
error instead of falling back).

**X11** (`x11_linux.bi`) goes through libxcb. xcb is small and has no
global state or locking callbacks. X servers on Linux number keys as evdev
code + 8, so physical keys need no keymap library. Auto-repeat shows up as
a release and a press with the same timestamp; a one-event lookahead
(`xcb_poll_for_queued_event`) turns that pair into `KeyDown(key, repeat: true)`.

**Wayland** (`wayland_linux.bi`, #18) speaks `xdg-shell` through
`libwayland-client.so.0`:

- **Protocol tables.** libwayland exports the core interfaces
  (`wl_compositor_interface` and the rest), found with `dlsym`. The
  `xdg-shell`, `xdg-decoration` and `text-input-unstable-v3` tables
  (`wl_interface`/`wl_message`, what `wayland-scanner`'s `private-code`
  generates) are written as Bismuth data in `wlproto_linux.bi` and built
  once per process into memory that is never freed.
- **Requests** go through `wl_proxy_marshal_flags`, which is variadic, so
  through a variadic C function pointer type
  (`extern c fn(*u8, u32, *WlInterface, u32, u32, ...) : *u8`, see the
  spec's C interop section). **Events** come to listeners: static tables of
  `export c fn` addresses, with the window's `State` as their data.
- **The loop.** `next()` pumps without blocking: `wl_display_prepare_read`,
  `wl_display_flush`, a zero-timeout `poll` on the display's fd, then
  `wl_display_read_events` (or `cancel_read`) and `dispatch_pending`.
- **Decorations.** Server-side through `zxdg_decoration_manager_v1` where
  the compositor offers it (KDE, wlroots). Elsewhere (GNOME, weston)
  `libdecor-0.so.0`, loaded at run time, makes the `xdg_surface` and
  toplevel and draws the frame with whatever plugin it finds; its
  configure, close and commit callbacks are `export c fn`s too, and
  `libdecor_dispatch` replaces our loop. `BISMUTH_LIBDECOR=0` turns it off;
  without either the window has no decorations.
- **Presentation.** A surface shows nothing until a buffer is attached.
  The window attaches a black `wl_shm` buffer (a `memfd`, `XRGB8888`) on
  each configure that changes its size, until `native()` hands the surface
  to a renderer: `gfx` makes a Vulkan surface with
  `VK_KHR_wayland_surface`, and its presents show the window from then on.
- **Sizes.** The first `xdg_toplevel.configure` with a size (a tiling or
  kiosk compositor's) becomes the window's size and a `Resize` event;
  `0x0` keeps the size asked for. Each `xdg_surface.configure` is acked at
  once. Focus is the toplevel's `activated` state. Sizes are the logical
  size times the buffer scale (see Content scale above).
- **Keys** are evdev codes straight from `wl_keyboard.key`. Wayland sends
  no repeats; the window repeats a held key itself at the compositor's
  `repeat_info` rate, checked on each pump. **Text**: after
  `w.textInput(true)`, `libxkbcommon.so.0` (loaded at run time) turns keys
  into `Text` events through the keymap the compositor sends, and the
  compositor's input method composes through `zwp_text_input_v3`,
  enabled while the window has focus: `preedit_string` and
  `commit_string` are applied at `done` (`Preedit`, then `Text`), and
  `w.setTextInputRect` is its `set_cursor_rectangle`. Without an input
  method, dead keys and `Multi_key` sequences compose through
  libxkbcommon's compose table (`xkb_compose`), as on X11.
- **The pointer**: a cursor from `libwayland-cursor`'s default theme
  (`XCURSOR_SIZE`); buttons from `BTN_LEFT`...; scrolling in wheel notches
  from `axis_discrete`, or the `axis` value over 10 for touchpads, summed
  per `wl_pointer.frame`.
- **Tests.** Headless compositors have no seat, so
  `tools/ci/check-gui.sh` runs `examples/window` under
  `weston --backend=headless` (with and without libdecor), and the event
  test `tests/gui/events_wayland.bi` in weston's X11 backend nested in
  Xvfb under the kiosk shell, where the test drives the X server with
  XTest and weston turns that into `wl_pointer` and `wl_keyboard` events.
  Hosts without weston skip both.

### macOS (Cocoa through the Objective-C runtime)

`std objc` loads `libobjc.A.dylib`. A message send is `objc_msgSend` cast
to the method's C type with the new function pointer types, for example
`objc.send<extern c fn(*u8, *u8, NSRect, u64, u64, i8) : *u8>()` for
`initWithContentRect:styleMask:backing:defer:`.

- `NSApplication sharedApplication` and `setActivationPolicy:Regular`, an
  `NSWindow`, and a delegate class built at run time
  (`objc_allocateClassPair` + `class_addMethod`) whose methods
  (`windowShouldClose:`, `windowDidResize:`, focus) are `export c fn`s.
  Events come from `nextEventMatchingMask:untilDate:inMode:dequeue:` on the
  main thread, which is where `main` already runs, inside an autorelease
  pool. Key events are not forwarded to `sendEvent:`, so the system does
  not beep at them.
- **Struct returns.** On x86-64, a method returning more than 16 bytes
  (`NSRect`) uses `objc_msgSend_stret` with the result pointer first.
  Bismuth's SysV lowering already passes the hidden result pointer first,
  so `objc.sendStret<extern c fn(*u8, *u8) : NSRect>()` is correct with no
  compiler change. `NSPoint` and `NSSize` (two doubles) return in XMM
  registers through plain `objc_msgSend`. On arm64 (#9) there is no
  `_stret`, so `objc.sendStret` will return `objc_msgSend` there.
- **Sizes.** The requested size is in points; reported sizes are pixels
  (points × `backingScaleFactor`).
- **App bundles** (#19). A bare executable can open a window, but a real
  app is a `.app` bundle. `bic build --app` writes one: `Info.plist` (bundle
  id, version, `NSHighResolutionCapable`), an `.icns` icon from `--icon`,
  and the executable signed ad hoc with the bundle bound in (Info.plist
  and `_CodeSignature/CodeResources` hashes in the CodeDirectory's special
  slots), as `codesign --sign -` does. Distribution needs Developer ID and
  notarization: `--sign-identity` on a Mac, then `notarytool` and
  `stapler`; see "GUI programs" in [tooling.md](tooling.md). Built and
  checked structurally; not yet run on a Mac (#26).

Untested so far: the macOS half and Metal compile and link, and bic
precompiles Metal libraries with Xcode where it runs on a Mac (#68,
[shaders.md §3.1](shaders.md#31-precompiled-binaries)), but none of it has
run on a Mac yet: the macOS runner (macbook) has been down (#26). See §7.

## 2. Renderer

### What others do

| | bgfx | sokol_gfx | SDL3 GPU | wgpu / WebGPU | Diligent | The Forge |
|---|---|---|---|---|---|---|
| Model | handles, view ids and 64-bit sort keys; state bits per draw; `submit(view, program)` | immutable buffer/image/shader/pipeline objects, passes, uniform blocks pushed per draw | explicit render/compute/copy passes, pipelines, transfer buffers, command buffers | device, queue, command encoder, render pass encoder, bind groups, pipelines | D3D12-like, with resource states tracked or explicit | thin D3D12/Vulkan-like layer with renderer-side descriptor sets |
| Threading | an encoder per thread, merged and sorted at frame end; a render thread | single-threaded | command buffers recorded on any thread | encoders on any thread | deferred contexts or command lists | command lists per thread |
| Backends | D3D11/12, Vulkan, Metal, GL/GLES, WebGPU, … | D3D11, Metal, GL, WebGPU (Vulkan in progress) | Vulkan, D3D12, Metal | Vulkan, Metal, D3D12, GL | D3D11/12, Vulkan, Metal (via MoltenVK), GL | D3D12, Vulkan, Metal, consoles |
| Shaders | shaderc: GLSL-like source to every backend's blob | sokol-shdc: GLSL to all, with reflection into a C header | precompiled per backend (SPIR-V/DXIL/MSL), SDL_shadercross | WGSL (naga), SPIR-V | HLSL to everything | an FSL macro language over HLSL |
| Cost per draw | low: sorting amortizes state changes | very low: no tracking, explicit pipelines | low | moderate: validation, tracking | moderate | very low |

What makes bgfx fast is CPU-side: no allocation per draw, state packed
into bits, draws batched and sorted by key so state changes are minimal,
and submission spread across threads through encoders. sokol_gfx gets
equal or better per-draw cost by making the expensive state immutable
(pipelines) and tracking nothing. WebGPU's model of bind groups and passes
maps one-to-one onto Vulkan, Metal and D3D12, but its validation and
resource tracking cost CPU time we don't want in release builds.

### Bismuth's API: sokol/WebGPU-shaped, with bgfx's encoders

The object kinds are `Device`, `Buffer`, `Image`, `Sampler`, `Shader`,
`Pipeline` (shader plus all fixed state: blend, depth, raster, vertex
layout, formats) and `Bindings` (a WebGPU bind group: buffers, images and
samplers for one draw). Each is created from a descriptor struct literal
with defaults and is immutable.

- **Handles** are generational 32-bit indices into per-kind pools, as in
  sokol: no reference counting, no hashing, and a use-after-destroy check
  that `--unchecked` compiles away.
- **Passes**: `beginPass(desc)` with clear/load/store per attachment,
  either into the swapchain or into images. Draws happen inside a pass;
  dispatches inside a compute pass (`beginCompute()`, slice 3).
- **Encoders**: `dev.encoder()` gives one per thread. It records
  `setPipeline`, `setBindings`, `setUniforms` (a per-frame ring buffer,
  like sokol's uniform blocks or bgfx's `setUniform`) and
  `draw`/`drawIndexed` into a compact Bismuth command stream.
- **Submission**: `commit()` translates each encoder's stream into native
  command buffers: Vulkan secondaries (or one primary per thread), D3D12
  command lists, Metal parallel render encoders, or a replay on D3D11's
  immediate context. It can do that on the `parallel for` pool and submit
  in encoder order. bgfx-style sort keys are an option per pass, not a
  requirement (`PassDesc.sort`, slice 3).
- **No hidden tracking.** Images declare their usage at creation; barriers
  and layout transitions come from pass boundaries, since every resource's
  state at a pass edge is known from the pass descriptor.
- **Frames in flight**: two, with per-frame staging and uniform rings.
  Dynamic buffers get `update` once per frame or `append` many times, as in
  sokol.

```bismuth
// shaders are Bismuth functions; core.shader embeds their SPIR-V, MSL and HLSL
// and their vertex and binding layouts at compile time (docs/shaders.md)
let pip = dev.pipeline({ vertex: core.shader(gfx.spriteVs), fragment: core.shader(gfx.spriteFs), blend: .Alpha })?;
let vb = dev.buffer({ usage: .Vertex, data: gfx.bytes(quad) })?;
let bind = dev.bindings({ vertex: [vb], images: [atlas], samplers: [linear] })?;
if dev.beginPass({ clear: gfx.rgb(0.0, 0.0, 0.0) }) {
    var enc = dev.encoder();
    ref enc.setPipeline(pip);
    ref enc.setBindings(bind);
    for s in sprites {
        ref enc.setUniforms(0, s.transform);
        ref enc.draw(0, 6);
    }
    dev.endPass();
}
dev.commit();
```

### The API, as built (#16)

| Call | What it does |
|---|---|
| `gfx.open(window, Options { backend, vsync, depth, debug })` | a device drawing into the window; `depth: true` gives the window's passes a `Depth32F` depth buffer, `debug: true` (or `BISMUTH_GFX_DEBUG=1`) the API's validation layer where installed |
| `gfx.headless(Options)` | a device with no window: passes draw into render target images, `readImage` reads them back (tests, tools, benchmarks; it needs no desktop session) |
| `dev.buffer(BufferDesc { usage: .Vertex/.Index, data, size, dynamic })` | immutable buffers take their data at creation; dynamic ones get `dev.update(buf, bytes)` once a frame, and grow when the data is larger |
| `dev.image(ImageDesc { width, height, format, data, renderTarget })` | `Rgba8`, `Bgra8`, `R8`, `Rgba16F`, `Rgba32F` and `Depth32F`, and [more](#what-an-engine-renderer-needs-602-and-after); `dev.updateImage(img, bytes)` replaces the pixels |
| `dev.sampler(SamplerDesc { filter: .Nearest/.Linear, wrap: .Clamp/.Repeat/.Mirror })` | and per-axis wraps, `Border`, separate filters, anisotropy and LOD limits [below](#what-an-engine-renderer-needs-602-and-after) |
| `dev.shader(code)`, `dev.pipelineWith(desc, vs, fs)` | compiled stages that pipelines can share; `dev.pipeline(desc)` makes its own from `desc.vertex` and `desc.fragment`. Stages load from the DXBC or Metal library bic precompiled where it could ([shaders.md §3.1](shaders.md#31-precompiled-binaries)); `dev.sourceCompiles()` counts those compiled from source |
| `gfx.loadShader(path)` | a `ShaderCode` from a `.bishader` package written by `bic shader` |
| `dev.pipeline(PipelineDesc { vertex, fragment, primitive, indexType, blend, cull, frontFace, depthCompare, depthWrite, colorFormat, depthFormat })` | everything but the bindings; formats default to the window's. `blendState`, `writeMask` and `alphaToCoverage`: [below](#what-an-engine-renderer-needs-602-and-after) |
| `dev.bindings(BindingsDesc { vertex, vertexOffsets, index, indexOffset, images, samplers })` | what draws read; a stage's k-th texture and sampler read `images[k]` and `samplers[k]` |
| `dev.beginPass(PassDesc { clear, action, color, depth, clearDepth })`, `dev.endPass()` | into the window, or into `color` (a render target image), or into `depth` alone (a depth-only pass); `depthAction` [below](#what-an-engine-renderer-needs-602-and-after) |
| `dev.encoder()` | an encoder for the open pass; one per thread |
| `enc.setPipeline`, `setBindings`, `setUniforms(slot, value)`, `setScissor(rect)`, `setViewport(rect)`, `draw(first, count, instances, firstInstance)`, `drawIndexed(count, firstIndex, baseVertex, instances, firstInstance)` | recorded into the encoder's command stream |
| `dev.commit()` | submits the frame and presents it |
| `dev.readImage(img)` | a render target's pixels; waits for the GPU |
| `dev.destroyBuffer` … `destroyBindings`, `dev.close()` | objects go once the frames in flight are done with them; `close` frees what is left |
| `dev.stats()` | the last frame's passes, encoders, draws and uniform bytes |

Notes on the choices:

- **Handles.** An id is `generation << 16 | slot`, generations from 1, so
  `0` is never an id and a destroyed object's handle is refused
  (`bindings`) or panics (in a recorded command) instead of reaching
  whatever reuses the slot.
- **Uniforms** (`setUniforms(slot, value)`) are copied into the encoder's
  stream when recorded, which keeps recording thread-local, and into the
  backend's per-frame ring when the pass is translated: one host-visible
  buffer with a region per frame in flight, blocks at 256-byte offsets.
  Vulkan binds them as `UNIFORM_BUFFER_DYNAMIC` descriptors with dynamic
  offsets, so a pipeline's descriptor set depends only on its layout and
  the bindings' images and samplers, and is made once and cached; D3D12
  binds root constant buffer views (a GPU address, no descriptor); D3D11
  maps one dynamic constant buffer per pass and binds 256-byte windows of
  it with `VSSetConstantBuffers1` (Direct3D 11.1); Metal sets buffer
  offsets. The slot's block goes to every stage of the pipeline that has
  that slot.
- **Encoders** record into a compact word stream (a draw is four or five
  32-bit words); `endPass` translates the pass's encoders, in the order
  they were made, into the frame's command buffer. Recording needs no
  lock; only `dev.encoder()` takes one.
- **Passes and barriers.** Images live in shader-read layout outside
  passes; a pass moves its render target to attachment layout and back,
  and the window's image to present at `commit`. Uploads (immutable
  buffers and images, `updateImage`) are copies recorded into the frame's
  command buffer ahead of the next pass, from a per-frame staging buffer.
- **Scissor and viewport** are dynamic state everywhere, as std.ui needs
  (§5).
- **Not yet:** workgroup memory in compute stages (it waits on the
  shaders, [shaders.md §5](shaders.md#5-later)), instanced vertex buffers
  with a step rate other than 1, conditional rendering on a query's
  result (bgfx's `setCondition`), and a pipeline
  cache on disk for Vulkan and Direct3D 12 pipelines (Direct3D's compiled
  HLSL and Metal's pipelines are cached).

### Slice 3 (#63)

| Call | What it does |
|---|---|
| `dev.computePipeline(ComputeDesc { compute })` | a pipeline that runs a `@compute` stage; destroyed with `destroyPipeline` |
| `dev.beginCompute()`, `enc.dispatch(x, y, z)`, `dev.endPass()` | a compute pass: encoders record dispatches of compute pipelines, x × y × z workgroups of the stage's `@compute` size |
| `BufferDesc { usage: .Storage }`, `BufferDesc.storage` | a storage buffer, zeroed when made without data; `storage: true` makes a vertex or index buffer one too (not on Direct3D 11). A dynamic storage buffer (`update` each frame) is read-only to shaders |
| `BindingsDesc.storage` | a stage's k-th storage parameter (`MutSlice<T>` or `Slice<T>`, in parameter order) reads `storage[k]`. Compute stages write `MutSlice<T>`; any stage reads `Slice<T>` (vertex pulling, say) |
| `dev.readBuffer(buf)` | a buffer's bytes, once the GPU has finished what was committed |
| `ImageDesc.mipLevels` (0: a full chain), `dev.generateMipmaps(img)`, `SamplerDesc.mipFilter` (`.None`, `.Nearest`, `.Linear`) | mipmapped images: levels below 0 are made from level 0 when the image is made with data and by `updateImage`, and by `generateMipmaps` for a render target a pass drew into. `sampleLevel` reads a level |
| `ImageDesc.samples`, `PipelineDesc.samples`, `PassDesc.resolve`, `Options.samples` | MSAA: multisampled render targets (2, 4 or 8 samples), resolved into a single-sample image at the end of a pass; the window's passes with `Options.samples` draw into a multisampled target resolved into the window. Pipelines default to the window's sample count; a pass's must match. Multisampled images are neither sampled nor read back |
| depth images in `BindingsDesc.images` | a `Depth32F` render target is sampled as a `Texture2D`, its depth in `.r` |
| `PassDesc.sort`, `enc.setSortKey(key)` | a sorted pass: draws go in order of sort key, then pipeline, then bindings, each with the state it was recorded with |
| `Options.parallel` (default on), `BISMUTH_GFX_THREADS=1` | a pass with several encoders and at least 1,024 draws is translated on the `parallel for` pool: Vulkan secondary command buffers, Direct3D 12 command lists, a Metal parallel render command encoder |
| `Options.timing`, `Stats.gpuMs`, `Stats.dispatches` | the GPU time of the latest frame the GPU has finished, from timestamp queries (Vulkan, Direct3D 12 and 11) or the command buffers' GPU times (Metal); -1 without |

How each part works:

- **Compute and storage buffers.** Compute pipelines share the pools,
  handles and bindings of render pipelines; their stage's bindings get
  the same treatment as a vertex stage's (uniform blocks through the
  ring, textures and samplers through the bindings). Barriers come from
  pass edges, as for images: on Vulkan a compute pass starts with a
  memory barrier from what came before (uploads, shader and attachment
  writes) and ends with one making its writes visible to vertex input,
  every shader stage and copies; dispatches in a pass are ordered by a
  barrier between them, so each sees what the ones before wrote, as in
  WebGPU. Direct3D 12 tracks a storage buffer's state (a render pass moves
  the buffers its bindings name to a read state before it starts; a
  compute pass moves each to UAV or read as a dispatch binds it, with a
  UAV barrier between dispatches); storage buffers are root SRVs and UAVs
  there, a GPU address each. Direct3D 11 and Metal order the dispatches
  themselves. On Direct3D 11 storage buffers are structured buffers, whose
  element stride is fixed when they are made: a storage buffer is made
  when a pass first binds it, with that binding's stride, and binding it
  with another stride later panics.
- **Mipmaps.** Vulkan blits each level from the one above
  (`vkCmdBlitImage`, linear where the format allows), Direct3D 11 calls
  `GenerateMips`, Metal a blit encoder's `generateMipmapsForTexture:`, and
  Direct3D 12, which has neither, draws each level from the one above
  with `mipVs`/`mipFs` (built-in shaders, [shaders.md §3](shaders.md#in-the-renderer-16)).
  A pass into a mipmapped render target draws level 0.
- **MSAA.** Vulkan resolves through dynamic rendering's resolve
  attachment, Metal with the `StoreAndMultisampleResolve` store action,
  Direct3D 11 and 12 with `ResolveSubresource` after the pass. A
  multisampled target is stored, not discarded, so a later pass can
  `.Load` it; it stays in its attachment layout between passes.
- **Depth images** rest in a shader-read state between passes, as color
  render targets do, and move to the depth attachment state for a pass;
  on Direct3D they are `R32_TYPELESS` with `D32_FLOAT` and `R32_FLOAT`
  views. The window's own depth buffer is not an image and is not
  sampled.
- **Sorted passes** are backend-independent (`std/gfx/sort.bi`): at
  `endPass` each draw is collected with the state in force when it was
  recorded (pipeline, bindings, uniform blocks, scissor, viewport), the
  draws are radix-sorted on (key << 32 | pipeline slot << 16 | bindings
  slot), stably, and written into new command streams that restate only
  what changes between draws. The backends translate those streams
  instead of the encoders, with nothing of their own; a pass whose draws
  are in order already is translated as recorded. There are as many
  streams as the pass had encoders, so parallel translation applies to
  sorted passes too. A draw recorded with no bindings keeps the previous
  draw's.
- **Parallel translation.** Each stream gets its own command buffer (or
  list, or subordinate encoder), its own range of the uniform ring, worked
  out before the threads start, and records on the `parallel for` pool;
  they run in stream order. On Vulkan the pass's descriptor sets are made
  under a lock (a draw takes it only when its pipeline or bindings
  change); Direct3D 12 splits the frame's command list around the pass
  and executes the lists in order at commit. Direct3D 11 replays on its
  immediate context, on one thread.
- **Pipeline cache.** Direct3D keeps the DXBC it compiled from HLSL
  between runs ([shaders.md §3.1](shaders.md#31-precompiled-binaries)).
  Metal keeps pipelines in an `MTLBinaryArchive` in the same cache
  folder (`BISMUTH_SHADER_CACHE`, else `~/Library/Caches/bismuth/shader-cache`,
  macOS 11 and later): a pipeline is made from the archive when it is
  there, else made and added, and the archive is written at `close`. MSL
  with no precompiled library is still compiled at load.

### What an engine renderer needs (#602 and after)

Claymore's renderer, ported from bgfx to std.gfx, needs the state bgfx
gives a draw. These calls add it; each keeps the earlier behaviour as its
default. `tests/gpu/gfxengine.bi` checks them on a headless device.

| Call | What it does |
|---|---|
| `PipelineDesc.blendState: BlendState { srcColor, dstColor, opColor, srcAlpha, dstAlpha, opAlpha }` | any blend equation, overriding the `blend` preset: `BlendFactor` (`Zero`, `One`, `SrcColor`, `OneMinusSrcColor`, `SrcAlpha`, `OneMinusSrcAlpha`, `DstColor`, `OneMinusDstColor`, `DstAlpha`, `OneMinusDstAlpha`, `SrcAlphaSaturated`, `Constant`, `OneMinusConstant`) and `BlendOp` (`Add`, `Subtract`, `ReverseSubtract`, `Min`, `Max`). `Blend.state()` is a preset's equation. `Blend.Alpha` writes alpha as `src + dst * (1 - a)`; bgfx's `BLEND_ALPHA` blends alpha by the source alpha too, which is `BlendState.alpha()` |
| `PipelineDesc.writeMask` | the channels written: `gfx.WRITE_RED`, `WRITE_GREEN`, `WRITE_BLUE`, `WRITE_ALPHA`, `WRITE_RGB`, `WRITE_ALL` (the default); 0 writes no color |
| `enc.setBlendConstant(color)` | what the `Constant` factors read, for the following draws; each pass starts with (0, 0, 0, 0). Sorted passes keep each draw's |
| `PipelineDesc.alphaToCoverage` | with MSAA, the fragment's alpha decides how many samples it covers (alpha-tested foliage) |
| `PassDesc { depth }` with no `color` | a depth-only pass (shadow maps): the depth image is the whole target, on every device; its pipelines have `colorFormat: .None`. A pass into the window with a depth image of its own is no longer possible (it never needed one: `Options.depth`) |
| `PassDesc.depthAction` | what a pass does with depth first (`Clear`, `Load`, `DontCare`); unset, it follows `action`, so a pass can keep depth and clear color, or the reverse |
| `PipelineDesc.depthBias`, `depthBiasSlope`, `depthBiasClamp` | depth bias for shadow casters: a constant in the depth format's smallest steps (an integer on Direct3D), a factor of the triangle's depth slope, and a clamp (0: none); off while both are 0 |
| `SamplerDesc.compare`, `DepthTexture2D.sampleCompare(SamplerCmp, uv, ref)` | comparison samplers for shadow maps: a sampler made with a `compare` other than `Never` compares a reference depth with the texels and filters the results (percentage-closer filtering); shaders read the depth image as a `DepthTexture2D` through a `SamplerCmp` ([shaders.md](shaders.md#built-ins)). Vulkan's `compareEnable`, Direct3D's comparison filters, Metal's `compareFunction` |
| `PassDesc.colors`, `PassDesc.clears`, `PipelineDesc.colorTargets: List<ColorTarget { format, blend, blendState, writeMask }>` | several color targets (up to 8, `MAX_COLOR_TARGETS`): a fragment stage returning a struct of `vec4`s writes them in field order, each target with its own clear color (missing ones use `clear`), format, blending and write mask. `color` and `colorFormat` stay the one-target spelling. Targets share a size and sample count; a pass with several resolves none yet. Vulkan dynamic rendering with several color attachments, `OMSetRenderTargets` with an array on Direct3D, Metal's `colorAttachments[i]` |
| `PixelFormat` `R16F`, `Rg16F`, `R32F`, `Rg32F`, `Rgba16`, `Rgba8Srgb`, `Bgra8Srgb`, `Depth16`, `Depth24Stencil8`, `R32U`, `Bc1`, `Bc3`, `Bc5`, `Bc7`; `dev.supports(format, renderTarget)` | more formats, where the device has them: `supports` asks it (Vulkan's format features, Direct3D's `CheckFormatSupport`, Metal's `isDepth24Stencil8PixelFormatSupported` and `supportsBCTextureCompression`), and `image` refuses one it lacks. sRGB images decode when sampled and encode when drawn into. Compressed images are whole 4x4 blocks, sampled only, their data rows of blocks (`gfx.rowBytes`, `rowCount`, `imageBytes`); their mip levels and `R32U`'s are not generated. `R32U` is neither filtered nor blended, and fragment stages (whose results are `vec4`) don't draw into it. `Depth24Stencil8`'s stencil is not used yet; AMD's Vulkan drivers lack it, so ask `supports` and fall back to `Depth32F` |
| `SamplerDesc { minFilter, magFilter, wrapU, wrapV, wrapW, border, maxAnisotropy, lodBias, minLod, maxLod }` | the rest of a sampler: filters and wraps apart (`filter` and `wrap` set them all), `Wrap.Border` with a `BorderColor` (transparent black, opaque black or white: what every API has), anisotropic filtering up to 16 (the device's limit), a LOD bias and the levels sampled. Metal has no LOD bias on a sampler, so `lodBias` does nothing there |
| `ddx`, `ddy`, `fwidth` in fragment stages | screen-space derivatives ([shaders.md](shaders.md#built-ins)), for normal maps' tangent frames, specular antialiasing and grid lines |
| `ImageDesc.kind: ImageKind` (`.D2`, `.Cube`, `.D2Array`), `ImageDesc.layers`, `TextureCube`, `Texture2DArray`, `PassDesc.layer` | cube maps (six square faces, +X, -X, +Y, -Y, +Z, -Z) and arrays of up to `MAX_LAYERS` (2048) 2D layers: their data is level 0 of each layer in turn, `updateImage` and `readImage` take and give every layer, and mip levels are made per layer. Shaders sample them as `TextureCube` (by direction) and `Texture2DArray` (by uv and layer) ([shaders.md](shaders.md#built-ins)); a draw that binds an image of another kind to one panics. A pass into a cube or array render target draws into its `PassDesc.layer` (a 2D target beside it, such as a shared depth image, draws into its only one). Layered images are not multisampled |
| `dev.updateImageRegion(img, ImageRegion { x, y, width, height, level, layer }, data, stride)`, `dev.copyImage(src, from, dst, to)`, `dev.readImageRegion(img, region)` | a rectangle of one level and layer (a width or height of 0 reaches the edge): written from rows `stride` bytes apart (bgfx's pitch; 0: tight), copied on the GPU between images of one format (or levels and layers of one image), or read back. Updates and copies are recorded ahead of the next pass and leave lower mip levels alone (`generateMipmaps` remakes them); `readImageRegion` waits, as `readImage` does. Compressed images' regions are whole blocks |
| `PipelineDesc.vertexLayout: VertexLayout { buffers, attrs }`, `VertexBufferLayout { stride, instance }`, `VertexAttr { location, buffer, offset, format }`, `VertexFormat` | vertex data laid out by another program (bgfx's `VertexLayout`, a model file) drawn as it is: each attribute the vertex stage reads (by location, its fields in order across its struct parameters) from any buffer slot and offset, in any format of its kind: `Float`…`Float4`, `Int`…, `Uint`…, `Unorm8x2`/`x4`, `Snorm8x2`/`x4`, `Uint8x2`/`x4`, `Sint8x2`/`x4`, the same in 16 bits, `Half2`, `Half4`, `Unorm1010102` (floats for `f32`/`vecN` fields, `Sint` for `i32`/`ivecN`, `Uint` for `u32`); a format with fewer components fills the rest with 0 and w with 1. Strides are multiples of 4; a stride of 0 gives every vertex the same element (bgfx's missing attributes). Unset, the stage's struct parameters are the layout, as before |
| `enc.setUniformBytes(slot, bytes)`, `@uniform(slot: n)` | a uniform block from bytes whose size is known at run time (bgfx's uniform buffers), and a block's slot chosen in the shader, so that a vertex and a fragment stage's blocks are filled apart. A draw panics when a block was given so few bytes that the GPU would read past their room in the ring |
| `draw(..., firstInstance)`, `drawIndexed(..., firstInstance)` | instances numbered from `firstInstance`: per-instance vertex buffers are read from that instance on (bgfx's instance data offsets). `@instanceIndex` counts from it on Vulkan and Metal; Direct3D's `SV_InstanceID` starts at 0 |
| `enc.setVertexBuffer(slot, buffer, offset)`, `enc.setIndexBuffer(buffer, offset)` | vertex and index buffers at byte offsets for the following draws, in place of the bindings' (bgfx's per-draw `setVertexBuffer`/`setIndexBuffer` and transient buffers), with no `Bindings` per offset; `setBindings` puts the bindings' back. Sorted passes keep each draw's |
| `BufferDesc.indirect`, `enc.drawIndirect(args, offset, count, stride)`, `enc.drawIndexedIndirect(...)`, `enc.dispatchIndirect(args, offset)`, `DrawArgs`, `DrawIndexedArgs`, `DispatchArgs` | draws and dispatches whose arguments are in a buffer (bgfx's indirect buffers), written by the CPU or by a compute pass before (GPU-driven culling, grass): the same layout on every API. A pass reads them when it draws, after the compute passes before it; `count` draws go one at a time (Vulkan's several-at-once needs a feature not every device has). Vulkan's `vkCmdDraw*Indirect` and `vkCmdDispatchIndirect` (the buffer gets `INDIRECT_BUFFER` usage, and barriers before passes cover `DRAW_INDIRECT`), Direct3D 12's `ExecuteIndirect` with a command signature per kind (the buffer in `INDIRECT_ARGUMENT`), Direct3D 11's `Draw*InstancedIndirect` and `DispatchIndirect` from a copy made for it (a structured buffer can't hold them there), Metal's indirect draw and dispatch calls |
| `ImageDesc.storage`, `StorageImage2D<T>` (`load`, `store`, `size`, `atomicAdd`), `MutSlice<T>.atomicAdd`, `gfx.storageTexel(format)` | images compute stages read and write (bgfx's `setImage` with `Access::ReadWrite`, GLSL's `imageLoad`/`imageStore`/`imageAtomicAdd`): a storage image is a D2 image of one level and one sample, `Rgba8`, `Rgba16F` or `Rgba32F` (a `StorageImage2D<vec4>`, stored to), `R32F` (`<f32>`) or `R32U` (`<u32>`, with `atomicAdd`), zeroed when made without data. A compute stage's storage images count with its textures and bind `images[k]`, which must be storage images of that texel (a dispatch panics otherwise); any stage may also sample one as a texture. `atomicAdd` adds to a texel, or to an element of a `MutSlice<u32>`/`<i32>` storage buffer, and returns the value before: counters and slot claims (Claymore's grass counts its blades in an `R32U` texel). Dispatches see what the ones before them stored, and passes after see what a compute pass stored ([shaders.md](shaders.md#built-ins)) |
| `dev.occlusionQuery() : Query`, `enc.beginQuery(q)`, `enc.endQuery()`, `dev.queryResult(q) : i64?`, `dev.destroyQuery(q)` | occlusion queries (bgfx's `createOcclusionQuery`, a draw submitted with one, and `getResult`): the samples of the draws between `beginQuery` and `endQuery` that pass the depth and stencil tests, null until the GPU has finished a frame that began the query (a frame or two after `commit`; `queryResult` never waits), then the latest such frame's count, which later frames that do not begin it keep. One query counts at a time in an encoder, in a render pass, and ends in the encoder it began in; a query begun several times in a frame (in several passes or encoders) counts their sum, up to `MAX_QUERY_BEGINS` (1024) begins a frame. In a sorted pass each draw counts into the query it was recorded in |
| `dev.readImageAsync(img, region)`, `dev.readBufferAsync(buf, offset, size)`, `dev.readbackReady(rb)`, `dev.takeReadback(rb)`, `dev.destroyReadback(rb)` | readback without a stall, as bgfx's `readTexture`: the copy is recorded into the frame and returns a `Readback` at once; once the frame is committed and the GPU has finished it (a frame or two later), `readbackReady` says so without waiting and `takeReadback` gives the bytes (waiting if they have not arrived) and lets it go |

How the backends do it: Vulkan's color blend attachment state and
multisample state, with the blend constants as dynamic state
(`vkCmdSetBlendConstants`); Direct3D 11's blend state object (its blend
factor passed to `OMSetBlendState`) and Direct3D 12's pipeline blend
description with `OMSetBlendFactor`; Metal's color attachment descriptor
(its write mask bits are the other way round) and
`setBlendColorRed:green:blue:alpha:`. Direct3D refuses color factors for
alpha, so there `SrcColor` in an alpha factor is `SrcAlpha`, which is
what it reads on the other APIs. A depth-only pass has no color
attachment (Vulkan dynamic rendering with no color, `OMSetRenderTargets`
with none, a Metal pass descriptor with only its depth attachment), and
depth bias is the rasterizer state's (Vulkan, Direct3D) or the encoder's
`setDepthBias:slopeScale:clamp:` when a pipeline is set (Metal).
Cube and array images are Vulkan images with array layers (cube ones
`CUBE_COMPATIBLE`) sampled through a `CUBE` or `2D_ARRAY` view, with a 2D
view of each layer for passes; Direct3D 11 and 12 texture arrays
(`TEXTURECUBE` for cubes) with `TextureCube` and `Texture2DArray` shader
resource views and a render target or depth view of each layer (Direct3D
12 draws each layer's mip levels with `mipArrayFs`); and Metal `Cube` and
`2DArray` textures, written and read a slice at a time, a pass setting its
attachments' `slice`.
Storage images are made with Vulkan's `STORAGE` usage and rest in the
`GENERAL` layout, where dispatches write them and draws sample them
(other images rest in `SHADER_READ_ONLY_OPTIMAL`), bound as
`STORAGE_IMAGE` descriptors; `vec4` ones are declared without a format,
so the device's `shaderStorageImageWriteWithoutFormat` is enabled where it
has it. Direct3D 11 gives them an unordered access view
(`BIND_UNORDERED_ACCESS`), set with `CSSetUnorderedAccessViews` at their
u register and unbound when the compute pass ends. Direct3D 12 makes them
`ALLOW_UNORDERED_ACCESS` with a UAV beside the SRV; a bindings object
with storage images puts their UAVs 16 places after the SRVs in its run
of descriptors, which the root signature's UAV ranges point at; a dispatch
moves an image it writes to `UNORDERED_ACCESS` (and one it samples back
to its read state), the UAV barrier between dispatches orders them, and
the end of the pass moves them back. Metal makes them with `ShaderWrite`
usage (and `ShaderAtomic` for `R32U`, macOS 14) and sets them as compute
textures.
Each begin of an occlusion query is a query of the backend's own,
numbered among its frame's begins: Vulkan's `OCCLUSION` query pool, with
a range per frame in flight reset as the frame starts, `PRECISE` where the
device has `occlusionQueryPrecise` (else a visible draw may count any
nonzero number), read with `vkGetQueryPoolResults` once the frame's fence
is signaled; a Direct3D 12 `OCCLUSION` query heap, resolved into a
readback buffer at the end of the frame and read once the fence passes
it; Direct3D 11 `OCCLUSION` query objects, read with `GetData`; and a
Metal visibility result buffer per frame in flight, set on every render
pass, counted into with `setVisibilityResultMode:offset:` (`Counting`) and
read once the frame's command buffer has completed. `queryResult` reads
the frames that are done without waiting; a frame slot about to be
reused is read first.
Region updates are copies from the staging memory (`vkCmdCopyBufferToImage`,
`CopyTextureRegion` from a placed footprint, a Metal blit from a staging
buffer) or `UpdateSubresource` with a box on Direct3D 11; copies are
`vkCmdCopyImage` (in the `GENERAL` layout within one image),
`CopySubresourceRegion` and `CopyTextureRegion`, and a blit encoder's
`copyFromTexture:...toTexture:`. A readback copies into host-visible
memory (a Vulkan host buffer, a Direct3D 12 readback-heap buffer, a
Direct3D 11 staging resource, a shared Metal buffer) and is ready when the
submission that copies it has finished: Vulkan counts its submissions and
asks the fences (`vkGetFenceStatus`), Direct3D 12 compares the fence's
completed value, Direct3D 11 maps the staging resource with
`MAP_FLAG_DO_NOT_WAIT`, and Metal reads the command buffer's `status`.


### Backends

| OS | Backend | Status |
|---|---|---|
| Linux | Vulkan 1.3, or 1.2 with `VK_KHR_dynamic_rendering` (`libvulkan.so.1`, xcb surface; Wayland surface with the Wayland window) | slice 3: draws, compute, mipmaps, MSAA, secondary command buffers, timestamps; 2 frames in flight |
| Windows | Vulkan (`vulkan-1.dll`, win32 surface) | slice 3 |
| Windows | Direct3D 12 (`d3d12.dll`, `dxgi.dll`; the first hardware adapter, then WARP) | slice 3: root signatures from the shaders' bindings, root CBVs for uniforms and root SRVs/UAVs for storage buffers, command lists per stream |
| Windows | Direct3D 11 (`d3d11.dll`, flip-model swapchain; hardware, then WARP; its layouts and slots are `std.os.windows.d3d11`'s) | slice 3, translated on one thread; the fallback that works on every Windows 10/11 machine, in CI and in VMs |
| macOS | Metal (`CAMetalLayer` on the content view) | slice 3, untested (no macOS runner, #26) |
| macOS | Vulkan through MoltenVK | possible through the same `vulkan.bi`, with `VK_EXT_metal_surface`; low priority |

Every API is loaded at run time (`dl`), and their functions are called
through C function pointer types: Vulkan's dispatch table from
`vkGetInstanceProcAddr`/`vkGetDeviceProcAddr`, COM methods by vtable slot
(`com.method<T>(obj, slot)`, with an `extern com fn` type for a method that
returns a struct), and Objective-C through `objc_msgSend`. A program
built with `gfx` starts on any machine, and `gfx.open` returns
`GfxError.Unavailable` with the reason when a driver is missing. Vulkan
code lives once in `std/gfx/vulkan.bi` and only the surface is per-OS.
`Backend.Auto` takes Vulkan, then D3D12, then D3D11 on Windows, and Metal
on macOS. `BISMUTH_GFX=vulkan|d3d12|d3d11|metal` overrides it.

Passes on Vulkan are dynamic rendering (`VK_KHR_dynamic_rendering`, core
in 1.3), so there are no `VkRenderPass`/`VkFramebuffer` objects and
pipelines name their attachment formats instead of a render pass.

## 3. Shaders

**Done (#17): shaders are written in Bismuth** and compiled by bic; the
design, the subset and the API the renderer uses are in
[shaders.md](shaders.md). The plan was staged, and stage 2 (precompiled
blobs written elsewhere) was skipped: the built-in shaders the renderer and
the GUI need (`std/gfx/shaders.bi`) are Bismuth from the start.

1. **First: none.** Clears need no shader.
2. **Planned for slice 2: precompiled per-backend blobs**, embedded as data: SPIR-V for
   Vulkan, MSL source for Metal (`newLibraryWithSource:` compiles it at
   load), and HLSL for D3D11, compiled at load by `D3DCompile` from
   `d3dcompiler_47.dll`, which ships with Windows 10+. `core.shader` now
   provides exactly these three, generated from Bismuth.
3. **Bismuth shaders (done).** A function marked as a shader stage, written
   in a subset of Bismuth: scalars, `vec`/`mat`, structs, fixed arrays and
   control flow, with no heap, strings or closures. bic compiles it:
   - to **SPIR-V** directly. It is a simple binary SSA format. (It is
     built from a shader IR made from the checked AST rather than from
     bic's SSA IR, which has already flattened vectors into byte offsets
     and loops into an unstructured CFG: [shaders.md §4](shaders.md#4-how-bic-compiles-them).)
   - to **MSL** and **HLSL** as text, which Metal and `D3DCompile`/`dxc`
     compile at load or build time.

   Its inputs, outputs, uniforms and bindings are ordinary Bismuth
   structs, so the vertex layout and bind layout come from the types: no
   reflection step and no mismatched C header (sokol-shdc's job). The same
   function runs on the CPU for tests, and the IR interpreter makes it
   debuggable. This is a large compiler project (#17).

SPIRV-Cross and other external toolchains are not needed at any stage.
Compiling at load would cost startup time on D3D and Metal, so bic
precompiles DXBC on Windows and Metal libraries on macOS into
`ShaderCode`, and D3D caches what it had to compile on disk
([shaders.md §3.1](shaders.md#31-precompiled-binaries)).

## 4. What "as fast as bgfx" means

Speed here is CPU cost per draw and per frame. Both libraries call the same
driver, so GPU time is the same for the same work. The benchmarks
(`bench/gfx`, #16) run the same scene through Bismuth's `gfx`
and through bgfx (built from source, release, same backend), on the same
machine:

| Benchmark | Scene | Metric | Target |
|---|---|---|---|
| draw-call throughput | bgfx's `17-drawstress`: N cubes, each its own draw with its own transform uniform | most draws per frame at 60 fps; CPU ms per 10k draws | within 10% of bgfx, or better |
| submission cost | 100k draws, no state change between them; then with a pipeline or bindings change every 1, 10 or 100 draws, in recorded order and in a sorted pass (slice 3; bgfx always sorts) | ns per recorded draw (encoder), ns per draw at commit | within 10% of bgfx |
| multithreaded submission | the same, recorded from 1, 2, 4 and 8 threads | speedup over 1 thread, of recording and of commit (slice 3 translates the encoders in parallel) | scales at least as well as bgfx's encoders |
| frame time | clear and present only; then a typical frame (1k sprites, 50 pipelines) | CPU frame time (median, p99) and present-to-present interval | same median, no worse p99 |
| resource churn | create and destroy 1k buffers and images a frame; 64 MB of dynamic uploads a frame | µs per create/destroy; upload GB/s | within 10% of bgfx |

How they run (#16, `bench/gfx`):

- `bench/gfx/gfxbench.bi` is std.gfx's side, built with `bic build -O 3`;
  `bench/gfx/bgfxbench.cpp` is bgfx's, built against a bgfx checkout
  (`build-bgfxbench.bat`) with drawstress's own shaders. Both run
  **headless** (std.gfx's `gfx.headless`, bgfx's headless mode) into a
  1280x720 RGBA8 target with a D32F depth buffer, so presentation is not
  measured and a CI runner's service session will do. bgfx runs
  single-threaded (`renderFrame` before `init`), so `bgfx::frame()`
  translates and submits on the calling thread, as `commit()` does.
- CPU timed with the high-resolution clock around recording and commit.
  std.gfx's record and CPU times start once the pass is open, because
  `beginPass` is where it waits for the GPU to free the frame slot; bgfx
  waits inside `frame()`, which therefore holds any GPU wait. The frame
  interval is comparable across both.
- Each case: `--warmup` seconds, then `--secs` seconds of frames, `--runs`
  times; a run's result is the median over its frames, and the reported
  value the median over runs. The defaults are the 2 s, 10 s and 5 runs
  above; the `bench-gfx` workflow (run by hand) uses 1 s, 5 s and 3 to fit
  a CI job.
- `.github/workflows/bench-gfx.yml` runs both on the Windows runner (torus)
  on each of Vulkan, D3D12 and D3D11, and std.gfx alone on the Linux
  runner; `bench/gfx/results.py` makes the tables. Results, with the bgfx
  commit, drivers and the cases' caveats, are in `bench/gfx/RESULTS.md`.
- GPU time (slice 3): every case but churn also reports `gpu_ms`, the
  GPU's time for a frame: std.gfx's from timestamp queries
  (`Options.timing`, `Stats.gpuMs`), bgfx's from its frame timer. On
  lavapipe the queries add about 11 µs to a frame's CPU time;
  `gfxbench --no-gpu-time` leaves them out.
- Not measured yet: d6, razer and the macbook (no bgfx built there, and
  the macOS runner is down, #26), and slice 3's sorted and parallel cases
  against bgfx on torus (#89).

## 5. GUI: immediate mode first

| | Immediate mode (Dear ImGui, microui, egui) | Native widgets (Win32/Cocoa/GTK, or a retained toolkit) |
|---|---|---|
| Fit for games and tools | the norm: debug UIs, editors, in-game menus | poor: a second event loop and render path beside the game's |
| Code | the UI is ordinary code each frame: `if ui.button("Save") { ... }` | widget trees, callbacks, state sync |
| Portability | one implementation on `gfx` for every OS | three native toolkits, or a large dependency |
| Look and accessibility | custom look; screen readers need work (AccessKit-style) | native look and accessibility for free |
| Cost | a few thousand draw vertices a frame, batched into a handful of draws | none on our side |

Recommendation: an **immediate-mode GUI written in Bismuth on `gfx`**
(`std ui`, #21). It suits Bismuth's audience and dogfoods the renderer. It needs
text: a TrueType rasterizer (an stb_truetype-style port) into an atlas,
text input events with IME, clipboard access, and DPI scale. Native
widgets stay out of std: an app that wants them can call Cocoa or Win32
through `objc` and C interop. Accessibility is a later layer that exports
the immediate-mode tree, as egui does with AccessKit.

### `std ui`: the first slice

`std.ui` (#21, `std/ui/`) draws on the GPU through `gfx` (`ui.gpu`,
`std/ui/gpu.bi`) and on the CPU (`cpu.bi`). It is renderer-independent on
purpose: a frame ends as plain data, and a small piece of std.ui hands
that data to `gfx`.

- **Context.** `ui.new()` returns a `Ui` handle (shared state, so widget
  calls need no `ref`): `if gui.button("Save") { ... }`. The state kept
  between frames is what Dear ImGui keeps: the hot, active and focused
  ids, window rectangles, z-order and scroll offsets, open tree nodes, and
  the text field being edited. Ids are FNV-1a hashes of labels under an id
  stack (windows, tree nodes, tables, `pushId`).
- **Input** comes from `std.window` events (`gui.feed(e)`) or from calls
  that tests make directly (`mouseMove`, `mouseDown`, `wheel`, `key`,
  `typeText`). Keys and typed text are queued in arrival order, so a text
  field applies "type, Backspace, type" in one frame as typed. An input
  method's `Preedit` shows underlined at the caret of the field being
  edited, and keys pressed while it composes are the input method's.
- **The window**: `gui.bindWindow(w)` gives text fields the system
  clipboard (Ctrl, or Super/Command, + X, C, V), and each `endFrame` turns
  the window's text input on while a field is edited, puts the input
  method's candidate window at the caret (`w.setTextInputRect`) and sets
  the pointer's shape the frame asked for (`IBeam` over text fields,
  `ResizeNWSE` over a window's grip). Without a window, `wantsTextInput()`,
  `textInputRect()` and `cursor()` say the same for the program.
- **Accessibility**: each frame builds a `window.AccessTree` with a node
  per widget (windows, panels, scrolling regions, tables, tab bars and
  open tree nodes hold theirs), which a bound window gets at `endFrame`;
  `accessTree()` returns it otherwise. A screen reader's requests
  (`Event.Access` through `feed`) act in the next frame: `Press` clicks
  the item (anything that goes through `interact` too), `Focus` starts
  editing a text field, `SetValue` sets a field's text or a slider's
  value, `Increment`/`Decrement` step a slider, `Expand`/`Collapse` open
  headers, tree nodes and windows. `accessItem(label, role, rect)` adds a
  node for an item of your own.
- **Layout**: lines stacking down a region, `sameLine`, equal `columns`,
  tables with headers, indentation for tree nodes, scrolling regions, and
  windows (title bar to drag, an arrow to collapse, a grip to resize,
  raised by a click, a scrollbar and the wheel). Metrics are pixels at
  scale 1; `setScale` multiplies them and bakes the font again.
- **Text**: a TrueType parser and an exact-area rasterizer in Bismuth
  (`font.bi`, `raster.bi`, after stb_truetype and font-rs: simple and
  composite glyphs, cmap formats 4 and 12, no hinting or kerning yet), a
  single-channel glyph atlas packed in shelves with a white texel for
  shapes (`atlas.bi`), UTF-8 throughout. The built-in font is a 15 KB
  Latin subset of Instrument Sans (SIL OFL), stored in `fontdata.bi`.
- **Output**: a `DrawData` per frame, as Dear ImGui's `ImDrawData`:
  `gfx.UiVertex` vertices (20 bytes: pixel position, atlas uv, `Rgba8`),
  32-bit indices, and commands, each a run of indices with one scissor
  rectangle and one texture. Every shape samples the atlas (texture 0),
  so commands change only with the scissor: a typical window adds two or
  three, and runs that share a scissor are merged across windows. Images
  (`gui.image`, `drawImage`) are the program's textures, by an id of its
  own: a command each, with uvs from 0 to 1 across the image, drawn by
  `ui.gpu` with the `gfx.Image` that `setTexture` gave the id (a texture or
  a render target, sampled bilinear) and by the CPU renderer from the
  `image.Image` that `renderWith`'s map gives it.
- **CPU reference renderer** (`cpu.bi`, `gui.render(frame, ref img)`):
  runs `gfx.uiVs` and `gfx.uiFs` on the CPU, snaps vertices to 1/16 pixel,
  fills by the top-left rule within the scissor rectangle, and blends
  straight alpha. It is what the tests check pixels against, and it is
  integer arithmetic where the GPU's rules are exact, so frames are the
  same bytes on every OS.

#### What it needs from slice 2, and the call sequence

The GPU path is one pipeline and one draw per command. Slice 2 (#16)
provides all of it, scissor rectangles included; the table is what it was
asked for:

| Need | For std.ui |
|---|---|
| A pipeline from `core.shader(gfx.uiVs)` and `core.shader(gfx.uiFs)` | vertex layout from `UiVertex` (stride 20: `Float2` at 0, `Float2` at 8, `Unorm8x4` at 16), triangle lists, no depth, **no culling** (both windings are drawn), blend `SrcAlpha, OneMinusSrcAlpha` for color and `One, OneMinusSrcAlpha` for alpha |
| Dynamic vertex and index buffers, written once a frame | `frame.vertices` and `frame.indices` as bytes; 32-bit indices; sizes change every frame, so `update` grows the buffer (per frame in flight) |
| `setUniforms` | `gfx.UiUniforms.forScreen(width, height)`: pixels to clip space |
| An image and its upload | the atlas, uploaded as RGBA8 with each coverage byte `a` as `(255, 255, 255, a)` (D3D11 has no channel swizzle, so not R8), again whenever `frame.atlasVersion` changes; the atlas only grows taller, and a later step can upload the changed rows only |
| A sampler | linear or nearest, clamped: glyphs sit on whole pixels and sample texel centers, so both give the same pixels |
| **Scissor rectangles**, set per draw | not in §2's encoder yet: `setScissor(rect)` as dynamic state (Vulkan `VK_DYNAMIC_STATE_SCISSOR` and `vkCmdSetScissor`, D3D11 `RSSetScissorRects` with a scissor-enabled rasterizer state, D3D12 `RSSetScissorRects`, Metal `setScissorRect:`), in framebuffer pixels, clamped to it |
| `drawIndexed(count, firstIndex)` | one per command, base vertex 0 |

```bismuth
// once
let pip = dev.pipeline({ vertex: core.shader(gfx.uiVs), fragment: core.shader(gfx.uiFs),
                         blend: .Alpha, cull: .None })?;
let vb = dev.buffer({ usage: .Vertex, size: 1 << 16, dynamic: true })?;
let ib = dev.buffer({ usage: .Index, size: 1 << 16, dynamic: true })?;
let smp = dev.sampler({ filter: .Linear, wrap: .Clamp })?;
var atlas = dev.image({ width: 1, height: 1, format: .Rgba8 })?;
var bind = dev.bindings({ vertex: [vb], index: ib, images: [atlas], samplers: [smp] })?;
var uploaded = 0;

// every frame
let frame = gui.endFrame();
if frame.atlasVersion != uploaded {
    let a = gui.atlas();           // or dev.updateImage(atlas, ...) while its size is the same
    dev.destroyBindings(bind);
    dev.destroyImage(atlas);
    atlas = dev.image({ width: a.width, height: a.height, format: .Rgba8, data: a.rgba() })?;
    bind = dev.bindings({ vertex: [vb], index: ib, images: [atlas], samplers: [smp] })?;
    uploaded = frame.atlasVersion;
}
dev.update(vb, gfx.bytes(frame.vertices));
dev.update(ib, gfx.bytes(frame.indices));
if dev.beginPass({ clear: gfx.rgb(0.1, 0.1, 0.1) }) {
    var enc = dev.encoder();
    ref enc.setPipeline(pip);
    ref enc.setBindings(bind);
    ref enc.setUniforms(0, gfx.UiUniforms.forScreen(f32(frame.width), f32(frame.height)));
    for c in frame.cmds {
        ref enc.setScissor(c.clip);       // as a gfx.Rect in pixels
        ref enc.drawIndexed(c.indexCount, firstIndex: c.indexOffset);
    }
    dev.endPass();
}
dev.commit();
```

That body is now std.ui's own: `var g = ui.gpu(dev)?` makes the pipeline
and sampler once, `ref g.upload(gui, frame)?` (before the pass) writes
the buffers, growing them in powers of two from 64 KiB, and uploads the
atlas when its version changes, and `g.draw(ref enc, frame)` records the
draws. `TestUIDrawGPU` (`tests/gpu/uidraw.bi`) draws frames with it on a
headless device, over a scene in the same pass and with the atlas and
buffers growing, and compares every pixel with the CPU renderer's.
Shapes put their vertices on the 1/16-pixel grid, Vulkan's minimum
subpixel precision and the CPU renderer's, so anti-aliased edges land in
the same place on GPUs with finer precision (lavapipe's is 1/256).
The pipeline state above is already checked: `TestUIFrameGPU` in
`tests/rt_gpu.bi` draws a std.ui frame (`tests/gpu/guiframe.bi`)
through Vulkan with it, using `tests/gpu/vkdraw.c` (blending and a
dynamic scissor per draw), and compares the pixels with
`ui.renderFrame`'s. It runs where a C compiler and a Vulkan driver are
installed (lavapipe will do) and skips elsewhere.

## 6. Compiler and toolchain prerequisites

| Need | Status |
|---|---|
| C function pointer types (`extern c fn(...) : R`), callable inside `unsafe`, from `core.bitcast` of an address; struct arguments and results with the target's C ABI | **done** (this slice) |
| Loading libraries and symbols at run time | **done**: `std dl` (`dlopen`, `LoadLibraryA`) |
| `bic build --gui` (PE GUI subsystem) | **done** |
| `objc_msgSend_stret` on x86-64 | no compiler change: SysV passes the hidden result pointer first, which is that entry point's convention (`objc.sendStret`) |
| C-callable functions taking or returning structs by value on Windows (`lowerParamsWin64`) | **done** (#20): for D3D12 debug callbacks, ImGui-style C callbacks and `qsort`-style APIs with struct values |
| Variadic C function pointer types (`extern c fn(*u8, u32, ...)`) | **done** (#18): SysV x86-64 (AL holds the vector register count), Win64 (variadic floats also in integer registers) and AAPCS64 Linux; Apple arm64 (variadics on the stack) waits for a Mach-O arm64 target |
| `.app` bundles with `Info.plist`, icon, signing; Windows `.rsrc` (manifest with DPI awareness, icon, version) | **done** (#19): `bic build --app`, `--icon`, `--app-*`; the bundle signature is checked structurally here and by `codesign --verify` once a Mac runs CI (#26) |
| Shaders in Bismuth: SPIR-V, MSL and HLSL emitters, a shader subset checker | **done** (#17): [shaders.md](shaders.md) |
| MSVC C++ methods returning structs (COM) pass the result pointer after `this` | **done** (#20): COM method pointer types, `extern com fn(*u8) : D3D12_CPU_DESCRIPTOR_HANDLE`, for D3D12's `GetDesc` and `GetCPUDescriptorHandleForHeapStart` |
| A miscompile found on the way: a `match` arm whose block ends in a bound error value, in a function returning a result, returned `Ok` | **done** (#22, fixed in #29): a block arm is converted to the match's type, so the error is returned as `Err`; `tests/run/matcharms.bi` covers it |

## 7. What runs where

| | Linux | Windows | macOS |
|---|---|---|---|
| `window` | Wayland: `check-gui.sh` runs `tests/gui/events_wayland.bi` where weston is installed (weston 13 checked: its X11 backend in Xvfb, kiosk shell, XTest input; with and without libdecor). X11/xcb: tested on Xvfb (keys, mouse, scroll, resize, close via `WM_DELETE_WINDOW`) | Win32: CI runs `tests/gui/events_windows.bi` on torus, posting real messages to the window | Cocoa: builds; not run (no macOS runner) |
| `gfx` drawing (#16) | Vulkan: `tests/gpu/gfxdraw.bi` on a headless device, its pixels against the CPU rasterizer and 12 checks, in `tests/runner.bi` and `check-gui.sh` where a Vulkan driver is installed (checked on lavapipe and the Linux runners' AMD iGPUs in the benchmark job); `examples/window`, `triangle` and `sprites` under Xvfb | Vulkan, Direct3D 12 and Direct3D 11: CI runs `gfxdraw.bi` headless on each on torus's AMD RX 6600 XT (it needs no desktop, so it runs in the runner's service session), then `examples/window`, `triangle` and `sprites` for 60 presented frames on each | Metal: builds; not run (#26) |
| `gfx` slice 3 (#63) | Vulkan: `tests/gpu/gfxmore.bi` (23 checks: compute dispatches read back against the CPU, dynamic storage buffers, vertex pulling from a compute pass's output, mipmaps, 4x MSAA resolves, sampled depth, sorted passes, parallel translation, timestamps) in `tests/runner.bi` and `check-gui.sh`, clean under the Khronos validation layer with synchronization validation on lavapipe; `examples/triangle --samples 4` under Xvfb | `gfxmore.bi` on each of Vulkan, Direct3D 12 and Direct3D 11 on torus, and `examples/triangle --samples 4` on each | Metal: builds; not run (#26) |
| packaging (#19) | n/a | CI runs `examples/window --app` (GUI manifest, icon, version information) for 60 D3D11 frames and reads the version and icon back through Windows | `Window.app`: builds; CI checks it with `codesign --verify --strict` and runs it on a Mac (not yet run) |

`tools/ci/check-gui.sh` builds `examples/window`, `triangle` and `sprites`
for every target, `examples/window` also with
`--gui`, and packaged with `--app` and the Bismuth icon for Windows and
macOS, on every CI host, then runs what the host can show. Windows
CI runners (and ssh) run in session 0, where windows and input work but
nothing can be presented: D3D11 swapchain creation fails with
`DXGI_ERROR_NOT_CURRENTLY_AVAILABLE` and so does the Vulkan swapchain.
In that case the script runs the frames in the logged-in user's desktop
session through `tools/ci/run-desktop.ps1`, which uses a temporary
scheduled task with an interactive logon. It skips them when nobody is
logged on.

## 8. Next slices

1. Renderer slice 2 (#16) is in: buffers, images, samplers, shaders,
   pipelines, bindings and draws; Vulkan dynamic rendering; D3D11 and
   Metal to match; D3D12; multithreaded encoders; pipelines from
   `core.shader`; headless devices; the triangle and sprite examples;
   the benchmarks against bgfx (`bench/gfx/RESULTS.md`).
2. Renderer slice 3 (#63) is in: compute passes and storage buffers,
   mipmaps, MSAA and resolves, sampling depth images, sorted passes,
   translating a pass's encoders in parallel (Vulkan secondary command
   buffers, D3D12 command lists, Metal parallel render encoders), GPU
   timestamps in the benchmarks, and Metal's pipeline archive (§2,
   "Slice 3"). Left for #89: running the benchmarks against bgfx on d6,
   razer and the macbook, and again on torus for the sorted and parallel
   cases; the lavapipe crash seen with Mesa 26.
3. `std ui` (#21): the context, layout, widgets, TrueType text, the
   CPU renderer and drawing through `gfx` (`ui.gpu`, `std/ui/gpu.bi`)
   are in (§5), and so are `std window`'s text input on X11, Win32 and
   Cocoa, IME composition (Wayland, Win32, Cocoa), the clipboard and
   cursor shapes, which std.ui uses through `bindWindow`; next are
   kerning and an accessibility tree.

Native Wayland (#18), `.app` bundles with Windows resources (#19),
struct-valued C callbacks on Windows (#20) and Bismuth shaders (#17, see
[shaders.md](shaders.md)) are in. What remains for Wayland is HiDPI
(buffer scale and fractional scale), preedit text and `xkb_compose`,
clipboard and cursor shapes, and a run on GNOME with libdecor's GTK
plugin.
