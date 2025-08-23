# sdk-c

**Official C SDK for CopperlineOS (Phase-0).**  
Thin, portable C headers and libs to talk to CopperlineOS services over **ports** (Unix domain sockets + optional FD passing). Provides high-level clients for **copperd** (timeline), **compositord** (Vulkan/KMS compositor), **blitterd** (2D ops), and **audiomixerd** (audio graph).

> TL;DR: a small C API that lets you create layers, move sprites, run Copper programs, blit pixels, and wire audio—without pulling in a giant IPC stack.

---

## Status

- Targets **Phase-0** (Linux-hosted) JSON protocol v0.  
- Blocking, event-driven design (integrates with `poll/epoll`).  
- License: **MIT OR Apache-2.0**.

---

## Layout

```
sdk-c/
├─ include/
│  ├─ cl_ports.h         # low-level port client (framing, JSON, FDs)
│  ├─ cl_compositor.h    # compositord client (layers/regs/dmabuf bind)
│  ├─ cl_copper.h        # copperd client (load/start/stop, subscribe)
│  ├─ cl_blitter.h       # blitterd client (rect_fill/copy/convert)
│  └─ cl_audio.h         # audiomixerd client (nodes/connect/start/stop)
├─ src/                  # library implementation
├─ examples/
│  ├─ overlay_sprite.c   # create layer + animate via copper
│  ├─ rect_fill.c        # fill/copy via blitter
│  └─ tone_to_device.c   # audio tone → device
└─ cmake/ or meson.build # build system files
```

Installed pkg-config name (planned): **`copperline-sdk-c`**.

---

## Install

Until packages are published, add this repo as a submodule or install locally:

```bash
git clone https://github.com/CopperlineOS/sdk-c
cd sdk-c
mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
sudo cmake --install .
```

This installs headers to your include prefix and a shared lib: `libcopperline_sdk_c.so` (name subject to change).

Using **pkg-config** from another project:

```bash
cc myapp.c $(pkg-config --cflags --libs copperline-sdk-c)
```

Or with **Meson**:

```meson
dep = dependency('copperline-sdk-c', required: true)
executable('myapp', ['myapp.c'], dependencies: [dep])
```

---

## Quick start: overlay a sprite and animate it

```c
#include "cl_compositor.h"
#include "cl_copper.h"
#include <stdio.h>

int main() {
  cl_comp_t* comp = cl_comp_connect("/run/copperline/compositord.sock");
  if (!comp) { fprintf(stderr, "comp connect failed\n"); return 1; }

  int layer = 0;
  if (cl_comp_create_layer(comp, &layer) != 0) return 1;

  // Bind a raw RGBA8 image for quick testing
  if (cl_comp_bind_image(comp, layer, "/tmp/sprite.rgba", 128, 128, "RGBA8") != 0) return 1;
  cl_comp_layer_params p = { .x=100, .y=360, .alpha=1.0f, .z=10, .visible=1 };
  if (cl_comp_set(comp, layer, &p) != 0) return 1;

  // Load a tiny Copper program: move right 4 px per vsync
  cl_copper_t* cop = cl_copper_connect("/run/copperline/copperd.sock");
  if (!cop) { fprintf(stderr, "copper connect failed\n"); return 1; }

  const char* prog =
    "{\"version\":1,\"program\":[\""
    "{\"op\":\"MOVE\",\"reg\":\"layer[1].x\",\"value\":100},\""
    "{\"op\":\"MOVE\",\"reg\":\"layer[1].y\",\"value\":360},\""
    "{\"op\":\"LOOP\",\"count\":-1,\"label\":\"loop\"},\""
    "{\"label\":\"loop\"},\""
    "{\"op\":\"WAIT\",\"vsync\":true},\""
    "{\"op\":\"ADD\",\"reg\":\"layer[1].x\",\"delta\":4},\""
    "{\"op\":\"IRQ\",\"tag\":\"tick\"},\""
    "{\"op\":\"JUMP\",\"label\":\"loop\"}\""
    "]}";

  int prog_id = 0;
  if (cl_copper_load(cop, prog, &prog_id) != 0) return 1;
  if (cl_copper_start(cop, prog_id) != 0) return 1;

  printf("Animating… press Ctrl+C to exit.\n");
  // In a real app, you might subscribe and read IRQ events here.
  pause();

  cl_copper_close(cop);
  cl_comp_close(comp);
  return 0;
}
```

Build:

```bash
cc overlay_sprite.c $(pkg-config --cflags --libs copperline-sdk-c)
```

---

## Quick start: blit a magenta rectangle

```c
#include "cl_blitter.h"
int main() {
  cl_blit_t* b = cl_blit_connect("/run/copperline/blitterd.sock");
  cl_surface_desc dst = {
    .kind = CL_SURF_FILE, .path = "/tmp/sprite.rgba",
    .w = 256, .h = 256, .format = "RGBA8", .stride = 1024
  };
  cl_rect rect = { .x = 0, .y = 0, .w = 256, .h = 256 };
  cl_color c = { .r = 255, .g = 0, .b = 255, .a = 255 };
  cl_blit_rect_fill(b, &dst, &rect, &c);
  cl_blit_close(b);
  return 0;
}
```

---

## Threading & events

- All client handles are **not** thread-safe by default; guard with your own locks if sharing.  
- Integrate with `poll/epoll`: each handle exposes a file descriptor you can watch to read **event streams** (e.g., `vsync`, `irq`, `stats`).  
- Requests are synchronous; non-blocking helpers are provided for advanced users.

---

## Error handling

- Functions return `0` on success, negative `CL_ERR_*` on failure (e.g., `CL_ERR_CONN`, `CL_ERR_ARG`, `CL_ERR_PROTO`).  
- Many functions accept an optional `cl_err_t*` to receive a structured error with a short message.

```c
cl_err_t err = {0};
if (cl_comp_set(comp, layer, &p, &err) != 0) {
  fprintf(stderr, "set failed: %s\n", err.msg);
}
```

---

## DMABUF helpers (optional)

If `libdrm` is available, helper functions can query format, stride, and modifiers for DMABUFs. File-backed buffers (`path`) exist only for demos/tests; prefer DMABUF FDs in real apps.

---

## Versioning & compatibility

- Targets **ports protocol 0** (JSON/NDJSON).  
- Services publish their protocol in `ping`; the SDK logs a warning on mismatches.  
- Public C API uses **semver**; expect `0.x` while protocols stabilise.

---

## Building from source (library)

```bash
rustup default stable   # only needed if parts are generated from Rust tools
sudo apt install build-essential cmake pkg-config libdrm-dev  # names vary by distro

git clone https://github.com/CopperlineOS/sdk-c
cd sdk-c && mkdir build && cd build
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
sudo cmake --install .
```

---

## Contributing

- Keep examples tiny and buildable with a bare `cc` + `pkg-config`.  
- Add tests to `tests/` mirroring JSON messages seen on the wire.  
- Breaking API changes require an RFC and deprecation notes.

See `CONTRIBUTING.md` and `CODE_OF_CONDUCT.md`.

---

## License

Dual-licensed under **Apache-2.0 OR MIT**.

---

## See also

- [`sdk-rs`](https://github.com/CopperlineOS/sdk-rs) – Rust SDK (async/blocking)  
- [`ports`](https://github.com/CopperlineOS/ports) – port protocol + low-level libs  
- [`copperd`](https://github.com/CopperlineOS/copperd) · [`compositord`](https://github.com/CopperlineOS/compositord) · [`blitterd`](https://github.com/CopperlineOS/blitterd) · [`audiomixerd`](https://github.com/CopperlineOS/audiomixerd)
