# GLFW Wayland patches for Minecraft

Patchset for [LWJGL-CI/glfw](https://github.com/LWJGL-CI/glfw) fixing several Wayland issues that affect Minecraft on Linux.

Tested on MC 1.21.11 + Fabric + KDE Plasma 6.3.6 with fractional scaling (1.75x) and NVIDIA.

## Patches

### 0001 - Key modifiers fix
Prevents `Ctrl+A` from inserting a literal 'a' in text fields while also triggering the select-all shortcut. Only fires `_glfwInputChar` when no modifier keys are held.

Based on [BoyOrigin/glfw-wayland](https://github.com/BoyOrigin/glfw-wayland) patch 0001.

### 0002 - glfwSetCursorPos with fallback for older compositors
The LWJGL-CI fork implements `glfwSetCursorPos` via the `pointer_warp_v1` protocol, which requires KDE 6.5+ or GNOME 49+. This patch adds a fallback using `zwp_locked_pointer_v1_set_cursor_position_hint` that works on older compositors when the pointer is locked (e.g. during Minecraft gameplay). Requests made while unlocked are deferred and applied once the lock is acquired.

Based on [BoyOrigin/glfw-wayland](https://github.com/BoyOrigin/glfw-wayland) patch 0003.

### 0003 - Fix window size callback not firing on unset fullscreen
When exiting fullscreen via `_glfwSetWindowSizeWayland`, the surface is resized but the size callback never fires. Clients like Minecraft keep the old fullscreen dimensions cached, causing input coordinate mismatches until a manual resize.

Based on [BoyOrigin/glfw-wayland](https://github.com/BoyOrigin/glfw-wayland) patch 0004.

### 0004 - Fix framebuffer size rounding with fractional scaling
The framebuffer dimensions are computed as `(size * scale) / 120`, which truncates. With fractional scaling (e.g. 1.75x), this can produce a buffer 1px smaller than the physical pixel area, causing a bilinear upscale and visible blur on every window. Uses round-half-up instead: `(size * scale + 60) / 120`.

Fix proposed by GamesTrap and mahkoh in [glfw#2713](https://github.com/glfw/glfw/issues/2713).

### 0005 - Implement cursor-shape-v1 protocol
Lets the compositor render standard cursors instead of the client uploading cursor pixmaps. Ensures the cursor theme, size and rendering are consistent between the application window and the rest of the desktop.

Based on [glfw#2679](https://github.com/glfw/glfw/pull/2679) by JakobDev.

## Prerequisites

Build dependencies (Wayland headers, protocol generators, X11 libs for dual support):
```bash
sudo apt install cmake ninja-build pkg-config libwayland-dev libxkbcommon-dev \
    wayland-protocols extra-cmake-modules libxrandr-dev libxinerama-dev \
    libxcursor-dev libxi-dev libgl-dev libwayland-bin
```


## Building

```bash
git clone https://github.com/LWJGL-CI/glfw
cd glfw
git am /path/to/patches/*.patch
cmake -B build -G Ninja \
    -DGLFW_BUILD_WAYLAND=ON \
    -DGLFW_BUILD_X11=ON \
    -DBUILD_SHARED_LIBS=ON \
    -DGLFW_BUILD_EXAMPLES=OFF \
    -DGLFW_BUILD_TESTS=OFF \
    -DGLFW_BUILD_DOCS=OFF
ninja -C build
```

The resulting `build/src/libglfw.so` can be used with Minecraft via JVM arguments:
```
-Dorg.lwjgl.glfw.libname=/path/to/libglfw.so
-Dsodium.checks.issue2561=false
```

The Sodium arg is required because Sodium runs a driver compatibility check at startup that incorrectly fails on Wayland with NVIDIA, preventing the game from launching.

Note: `-Dorg.lwjgl.librarypath` does NOT work here. LWJGL extracts its bundled `.so` from the jar first and ignores the library path. `glfw.libname` forces a specific file.

No wrapper command needed. The patched GLFW is based on 3.4+ which prefers native Wayland over XWayland automatically when `XDG_SESSION_TYPE=wayland` is set (default on any Wayland session).

## Companion mod: WayFix

These GLFW patches work alongside a patched [WayFix](https://github.com/jdkeke142/WayFix/tree/wayland-fixes) mod that fixes:
- Multi-monitor fullscreen targeting with CWB borderless (mixin priority fix)
- Click offset after fullscreen transitions with fractional scaling (per-frame size reconciliation)

WayFix uses [kdotool](https://github.com/jinliu/kdotool) on KDE Plasma to query the real window position from KWin (since Wayland doesn't expose global coords to clients). Without it, WayFix can only fall back to a manually configured monitor name. Install it for full multi-monitor support.
