# SDL3 fork with `xx_fractional_scale_v2` support

This tree is upstream SDL **3.4.20** (`release-3.4.20`, commit `f00dc91`) with a
single patch that makes Wayland surfaces use a **pixel** coordinate space when
the compositor provides KDE's private `xx_fractional_scale_v2` protocol.

It exists because Minecraft 26.3 dropped GLFW for SDL3, and at 150% scaling the
game is blurry again: SDL only speaks `wp_fractional_scale_v1`, so it renders a
2560x1600 buffer but sets `wp_viewport.set_destination(1707, 1067)`. KWin maps
1707 logical pixels to `round(1707 * 1.5) = 2561` device pixels and resamples
the 2560 wide frame, which softens everything and shows a duplicated column at
the seam. Direct scanout is not 1:1 either (the plane is scaled 2560 -> 2561).

## The patch

`xx_fractional_scale_v2` lets a client *declare* the coordinate space its
surface acts in (`set_scale_factor`, 8.24 fixed point). Once the client declares
pixels, the compositor:

- multiplies `xdg_toplevel.configure` by the scale, so the client is asked for
  the exact device pixel size (2560x1600 instead of 1707x1067),
- divides `wp_viewport.set_destination` by the scale, so the client can pass the
  pixel size and the logical size becomes exactly 2560 / 1.5 = 1706.666...,
- sends pointer/touch/IME/drag coordinates in pixels.

That makes `buffer size == viewport destination == device size`, so the frame is
never resampled.

### Design in SDL terms

Reproducing GLFW's fork would mean converting every logical size to pixels at
the protocol boundary. SDL already has a pixel-based mode (`scale_to_display`),
so the patch instead makes the Wayland backend live in **client pixel space**
while `client_scale` is non-zero:

| thing | value under `xx_fractional_scale_v2` |
| --- | --- |
| `GetWindowScale()` (points <-> pixels helper) | 1.0 |
| backbuffer (`GetBufferSize`) | `requested.logical_*` (the configure is in pixels) |
| exclusive fullscreen backbuffer | the fullscreen mode size, i.e. 2560x1600 |
| `wp_viewport.set_destination` | the backbuffer size (pixels) |
| `data->current.logical_*` | forced equal to `current.pixel_*` (surface coords) |
| `pointer_scale` (surface -> SDL window coords) | 1.0 |
| `SDL_GetWindowSize()` / `SDL_GetWindowSizeInPixels()` | the same pixel size |
| `SDL_GetWindowDisplayScale()` | still 1.5, from the scale event |

So a window that asks for 854x480 is 854 physical pixels (569 logical points) at
150%, exactly like a 1:1 DPI-aware client. Apps that assume logical window
units will see smaller windows at fractional scales; that is the semantic cost
of the protocol and it does not affect Minecraft, which runs exclusive
fullscreen at the native mode.

`Wayland_SeatWarpMouse` needs one extra conversion: KWin interprets
`wp_pointer_warp_v1` positions in compositor logical coordinates (it does not
apply the v2 client scale there, which is also why GLFW had to drop its own
multiply), so SDL divides the warp position by `client_scale` on top of the
usual `pointer_scale` conversion.

### Files

- `wayland-protocols/xx-fractional-scale-v2.xml`: the protocol description
  (copied from KWin); the CMake build globs `*.xml`, so `wayland-scanner` picks
  it up automatically.
- `src/video/wayland/SDL_waylandvideo.[ch]`: bind/destroy
  `xx_fractional_scale_manager_v2`.
- `src/video/wayland/SDL_waylandwindow.[ch]`: per-surface
  `xx_fractional_scale_v2` object and listener, the pixel-space backbuffer,
  viewport, opaque region and pointer scale decisions, and the scale priority
  guards (`wp_fractional_scale_v1` is only used when v2 is unavailable).
- `src/video/wayland/SDL_waylandmouse.c`: warp conversion described above.

When the compositor does not advertise
`xx_fractional_scale_manager_v2`, the patch is inert: `client_scale` stays 0 and
SDL behaves exactly like upstream 3.4.20.

## Build

```sh
cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=Release \
      -DSDL_SHARED=ON -DSDL_STATIC=OFF -DSDL_TESTS=OFF -DSDL_EXAMPLES=OFF
cmake --build build --target SDL3-shared
```

## Using it

KWin (Plasma 6.8) must be started with

```sh
KWIN_WAYLAND_SUPPORT_XX_FRACTIONAL_SCALE_V2=1
```

otherwise the global is not advertised and nothing changes. Check with
`wayland-info | grep xx_fractional_scale`.

LWJGL loads SDL through its bundled natives, so point it at the patched build
(either one is enough):

```sh
export LD_LIBRARY_PATH=/home/bot/lwjgl/sdl/build:$LD_LIBRARY_PATH
java -Dorg.lwjgl.sdl.libname=/home/bot/lwjgl/sdl/build/libSDL3.so ...
```

The Minecraft 26.3 launcher in `~/xx-fractional-scale-v2-test/start-26.3.sh`
does both.

## Verified

`WAYLAND_DEBUG=1` trace of Minecraft 26.3 at 150% (2560x1600 output):

```
-> xx_fractional_scale_v2#64.set_scale_factor(25165824)      # 1.5 in 8.24
xdg_toplevel#72.configure(2560, 1600, array[4])              # pixels, was 1707x1067
-> wp_viewport#63.set_destination(2560, 1600)                # == buffer, was 1707x1067
-> zwp_linux_buffer_params_v1#55.create(2560, 1600, ...)     # backbuffer
```

`~/lwjgl/sdl-v2-check.c` is a small standalone test of the same path
(windowed, `--fullscreen`, `--exclusive`).

## Limitations / open points

- Mouse and keyboard behaviour in game are not covered by the trace; warping is
  the only input path with an extra conversion.
- SDL's client-side decorations and xdg popups are untested under v2. Their
  surface coordinates are pixels as well, so they should line up, but Minecraft
  is fullscreen and does not use them.
- Windowed apps see window sizes in pixels at fractional scales (see above).
