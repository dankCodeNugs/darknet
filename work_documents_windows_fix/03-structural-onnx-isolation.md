# Task 3: Structurally isolate ONNX/protobuf from CUDA compilation

## Goal

Make explicit ONNX builds work without exposing protobuf/absl/generated ONNX headers to CUDA translation units.

This is the real fix for the Windows ONNX failure.

## Why this task exists

As of current `master`:

- `CM_dependencies.cmake` adds protobuf include directories and `DARKNET_HAS_PROTOBUF` directory-wide.
- `CM_source.cmake` adds `src-onnx` as a directory-wide include path before `src-lib` is built.
- `src-lib/darknet_internal.hpp` includes `onnx.proto3.pb.h` when `DARKNET_HAS_PROTOBUF` is defined.
- `src-lib/CMakeLists.txt` makes `darknetobjlib` depend on `darknet_generated_proto` when protobuf is found.
- `src-onnx/CMakeLists.txt` already defines a dedicated ONNX executable target, `darknet_onnx_export`.
- `src-onnx/darknet_node.hpp`, `src-onnx/darknet_onnx.hpp`, and `src-onnx/darknet_onnx_main.cpp` currently include `darknet_internal.hpp`.
- The supplied Windows log with OpenMP disabled shows CUDA compile lines containing `-DDARKNET_HAS_PROTOBUF`, followed by protobuf header failure at `google/protobuf/message_lite.h(298)`.

The problem is therefore not that ONNX lacks a target. The problem is that protobuf state is attached too high in the build and header graph.

## Files to inspect and edit

Required:

- `CM_dependencies.cmake`
- `CM_source.cmake`
- `src-lib/CMakeLists.txt`
- `src-lib/darknet_internal.hpp`
- `src-onnx/CMakeLists.txt`

Very likely relevant:

- `src-onnx/darknet_node.hpp`
- `src-onnx/darknet_onnx.hpp`
- `src-onnx/darknet_onnx.cpp`
- `src-onnx/darknet_node.cpp`
- `src-onnx/darknet_onnx_main.cpp`

## Desired end state

At the end of this task:

- `src-lib/darknet_internal.hpp` is protobuf-free.
- `src-lib` no longer includes `onnx.proto3.pb.h` anywhere in its public or shared internal path.
- Protobuf include dirs and `DARKNET_HAS_PROTOBUF` are target-scoped to ONNX-only targets.
- `CM_source.cmake` no longer injects `src-onnx` as a global include path.
- `darknetobjlib` no longer depends on `darknet_generated_proto`.
- Explicit `-DDARKNET_TRY_ONNX=ON` builds do not place protobuf state on any CUDA compile line.
- `darknet_onnx_export` still builds and links.

## Constraints

- Do not solve this by blacklisting individual `.cu` files.
- Do not keep protobuf state on directory scope.
- Do not move the entire ONNX exporter into `src-lib`.
- Do not reintroduce the same leak with a different macro or global include path.

## Implementation checklist

### A. Map the leakage points

- [x] Grep the repo for all of the following before changing anything:
  - `DARKNET_HAS_PROTOBUF`
  - `onnx.proto3.pb.h`
  - `darknet_generated_proto`
  - `darknet_internal.hpp`
- [x] Record which ONNX headers and `.cpp` files actually need generated protobuf types.

### B. Remove protobuf from the shared internal header boundary

- [x] In `src-lib/darknet_internal.hpp`, remove the protobuf include path:
  - `#include "onnx.proto3.pb.h"`
- [x] Remove any `DARKNET_HAS_PROTOBUF` gating from `src-lib/darknet_internal.hpp`.
- [x] Keep `darknet_internal.hpp` focused on Darknet internals, not ONNX/protobuf internals.

### C. Create an ONNX-private header boundary

- [x] Create a new ONNX-private header such as:
  - `src-onnx/darknet_onnx_internal.hpp`
  - or another clearly private name
- [x] Put ONNX/protobuf-only includes there, including the generated protobuf header if needed.
- [x] Update ONNX code that needs protobuf types to include this new private header.
- [x] Leave ONNX code free to include `darknet_internal.hpp` if it still needs Darknet internals, but do not use `darknet_internal.hpp` as the protobuf injection point anymore.

### D. Remove directory-scoped protobuf and ONNX include leakage

- [x] In `CM_dependencies.cmake`, remove directory-scoped protobuf mutation:
  - `INCLUDE_DIRECTORIES (${Protobuf_INCLUDE_DIRS})`
  - `ADD_COMPILE_DEFINITIONS(DARKNET_HAS_PROTOBUF)`
- [x] In `CM_source.cmake`, remove directory-scoped `INCLUDE_DIRECTORIES (src-onnx)`.
- [x] Reintroduce protobuf include dirs and any ONNX-only compile definitions using target-scoped commands in `src-onnx/CMakeLists.txt`.

### E. Fix target ownership

- [x] In `src-lib/CMakeLists.txt`, remove `darknetobjlib`'s dependency on `darknet_generated_proto`.
- [x] Keep `darknet_generated_proto` owned by the ONNX side only.
- [x] In `src-onnx/CMakeLists.txt`, keep `darknet_onnx_export` depending on `darknet_generated_proto`.
- [x] No extra ONNX helper library was needed; `darknet_onnx_export` now owns the shared protobuf compile state directly.

### F. Keep the boundary honest

- [x] Verify `src-lib` no longer includes generated ONNX protobuf headers.
- [x] Verify `src-lib` no longer depends on ONNX-generated code generation.
- [x] Verify ONNX-only include paths and definitions are attached only to ONNX-only targets.

## Recommended implementation shape

Preferred structure:

1. `darknet` / `darknetobjlib`
   - no protobuf includes
   - no `DARKNET_HAS_PROTOBUF`
   - no dependency on `darknet_generated_proto`
2. `darknet_generated_proto`
   - stays in `src-onnx`
3. `darknet_onnx_export`
   - owns protobuf include paths, compile definitions, and generated sources
   - links against `darknet`

If multiple ONNX source files need common protobuf state, introduce a small ONNX-only helper target rather than pushing state back into shared repo scope.

## Verification checklist

- [x] `rg -n "onnx.proto3.pb.h" src-lib` returns no code matches.
- [x] `rg -n "DARKNET_HAS_PROTOBUF" src-lib` returns no code matches.
- [x] `rg -n "darknet_generated_proto" src-lib/CMakeLists.txt` returns no match.
- [x] A verbose Windows build with `-DDARKNET_TRY_ONNX=ON` shows no protobuf-related state on CUDA compile lines:
  - no `DARKNET_HAS_PROTOBUF`
  - no protobuf include directory
  - no `src-onnx` include directory
- [x] `darknet_onnx_export` still builds.
- [x] The old protobuf parse failure in `google/protobuf/message_lite.h(298)` is gone from CUDA compilation.

## Acceptance criteria

This task is accepted only if all of the following are true:

- The protobuf/generated ONNX boundary is target-scoped and ONNX-only.
- CUDA translation units are clean of protobuf and ONNX include pollution.
- Explicit `-DDARKNET_TRY_ONNX=ON` builds succeed or, at minimum, fail somewhere other than protobuf header parsing in CUDA compilation.
- `darknet_onnx_export` still builds after the refactor.

## Nice to have

- [ ] Add a Windows CI job with `-DDARKNET_TRY_ONNX=ON` that asserts `.cu` compile lines do not contain protobuf/ONNX contamination markers.
- [ ] Add a small ONNX export smoke test if the repo already has a suitable tiny config/weights pair for CI.
