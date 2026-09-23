# goamdsmi

Go bindings for AMD SMI (`package goamdsmi`), repackaged as a standalone,
`go get`-able module.

## Why this repo exists

The upstream binding lives at `projects/amdsmi/` inside the ROCm monorepo, which
ships **no `go.mod`** at that subdirectory. As a result:

```
$ go get github.com/ROCm/rocm-systems/projects/amdsmi@develop
go: github.com/ROCm/rocm-systems/projects/amdsmi@develop: create zip:
    module source tree too large (max size is 524288000 bytes)
```

Go falls back to the whole monorepo as one module and hits its 500 MB zip cap.
This repo extracts just the Go binding (+ its C shim source) into a repo-root
module so it can be fetched normally.

## Provenance

Extracted from:

- repo:   https://github.com/ROCm/rocm-systems
- path:   `projects/amdsmi`
- branch: `develop`
- commit: `820ea79c1848e1291204e7f7e56ec68bd049704e`

Included: `goamdsmi.go`, `goamdsmi_shim/`, `LICENSE`.

### Local patches

- `goamdsmi.go` (`GO_gpu_uma_carveout_info_get`): upstream casts `options` to
  `*[16][256]C.char`, but the C parameter `char options[][256]` decays to
  `char (*)[256]`, which cgo types as `*[256]C.char`. Upstream therefore fails
  to compile under cgo:

  ```
  goamdsmi.go:740:3: cannot use (*[16][256]_Ctype_char)(unsafe.Pointer(options))
      (value of type *[16][256]_Ctype_char) as *[256]_Ctype_char value
  ```

  Fixed by casting to `*[256]C.char` (same memory, pointer to first row).
  Go API is unchanged.

- `goamdsmi_shim/smiwrapper/amdsmi_go_shim.c`: upstream develop targets
  amd-smi 27.x and does not compile against amd-smi 26.x (ROCm 7.1 / 7.2).
  Added compile-time guards so the shim builds against both:
  - `AMDSMI_LIB_VERSION_MAJOR < 27`: alias `amdsmi_processor_type_t` to the
    old name `processor_type_t`, and declare
    `amdsmi_get_processor_handles_by_type()` (exported by libamd_smi 26.x, but
    only declared under `ENABLE_ESMI_LIB`).
  - UMA carveout / TTM wrappers are compiled only if the header defines
    `AMDSMI_MAX_CARVEOUT_OPTIONS`; otherwise they return `-1`.

  Verified: builds against ROCm 7.1.1 and 7.2.4 (amd-smi 26.2) and
  syntax-checks against the develop header (amd-smi 27.1).

## Install

```
go get github.com/ichbinblau/goamdsmi@latest
```

## Usage

```go
import goamdsmi "github.com/ichbinblau/goamdsmi"

if !goamdsmi.GO_gpu_init() { /* handle */ }
defer goamdsmi.GO_gpu_shutdown()
n := int(goamdsmi.GO_gpu_num_monitor_devices())
```

## Build prerequisite (cgo shim)

This is a cgo wrapper that links `libgoamdsmi_shim64.so`, which in turn calls
`libamd_smi.so`. Regular ROCm installs do **not** ship the shim; build it from
`goamdsmi_shim/` (needs gcc and ROCm with amd-smi headers). Run this inside
your module after `go get`, so the shim source matches the Go binding version:

```bash
SRC=$(go list -m -f '{{.Dir}}' github.com/ichbinblau/goamdsmi)/goamdsmi_shim/smiwrapper
SHIM=$HOME/goamdsmi-shim
mkdir -p $SHIM/include $SHIM/lib
gcc -shared -fPIC -O2 -DENABLE_DEBUG_LEVEL=0 \
    -o $SHIM/lib/libgoamdsmi_shim64.so $SRC/amdsmi_go_shim.c \
    -I$SRC -I/opt/rocm/include -L/opt/rocm/lib -lamd_smi -Wl,-rpath,/opt/rocm/lib
cp $SRC/amdsmi_go_shim.h $SRC/goamdsmi.h $SHIM/include/
```

Then build/run your program with:

```bash
export CGO_ENABLED=1
export CGO_CFLAGS="-I$HOME/goamdsmi-shim/include"
export CGO_LDFLAGS="-L$HOME/goamdsmi-shim/lib"
export LD_LIBRARY_PATH="$HOME/goamdsmi-shim/lib:$LD_LIBRARY_PATH"
```

(If you install the header into `/opt/rocm/include` and the library into
`/opt/rocm/lib`, `CGO_CFLAGS`/`CGO_LDFLAGS` are not needed.)

A complete working example: https://github.com/ichbinblau/amdsmi-demo-A

## License

MIT (AMD). See [LICENSE](./LICENSE). This repository redistributes a subset of
`rocm-systems/projects/amdsmi`, with the patches listed above.
