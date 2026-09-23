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

`goamdsmi_shim/` is unmodified.

## Install

```
go get github.com/ichbinblau/goamdsmi
```

## Usage

```go
import goamdsmi "github.com/ichbinblau/goamdsmi"

if !goamdsmi.GO_gpu_init() { /* handle */ }
defer goamdsmi.GO_gpu_shutdown()
n := int(goamdsmi.GO_gpu_num_monitor_devices())
```

## Build prerequisite (cgo)

This is a cgo wrapper linking `libgoamdsmi_shim64.so`. Building/running requires,
from a ROCm install (or built from `goamdsmi_shim/` against the full amdsmi lib):

- `/opt/rocm/include/amdsmi_go_shim.h`
- `/opt/rocm/lib*/libgoamdsmi_shim64.so`

```
export CGO_ENABLED=1
export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/opt/rocm/lib:/opt/rocm/lib64
```

## License

MIT (AMD). See [LICENSE](./LICENSE). This repository redistributes a subset of
`rocm-systems/projects/amdsmi`, with the patch listed above.
