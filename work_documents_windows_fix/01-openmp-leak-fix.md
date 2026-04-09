# Task 1: Fix the Windows OpenMP leak into CUDA compilation

## Goal

Stop Windows OpenMP settings from leaking into nvcc compilation of `.cu` files.

This task is complete when Windows CUDA compile lines no longer contain `/openmp:experimental` or `DARKNET_OPENMP`, and the build advances beyond the current nvcc fatal.

## Why this task exists

As of current `master`:

- `CM_dependencies.cmake` uses directory-scoped OpenMP settings:
  - `ADD_COMPILE_DEFINITIONS (DARKNET_OPENMP)`
  - `LIST (APPEND DARKNET_LINK_LIBS OpenMP::OpenMP_CXX OpenMP::OpenMP_C)`
  - `ADD_COMPILE_OPTIONS (/openmp:experimental)` on Windows
- `src-lib/darknet_internal.hpp` includes `<omp.h>` whenever `DARKNET_OPENMP` is defined.
- The supplied Windows build log shows nvcc command lines for `.cu` files containing `/openmp:experimental` and `-DDARKNET_OPENMP`, then failing with:
  - `nvcc fatal: A single input file is required for a non-link phase when an outputfile is specified`

## Files to inspect and edit

Required:

- `CM_dependencies.cmake`
- `src-lib/CMakeLists.txt`

Likely relevant:

- `src-lib/darknet_internal.hpp`
- `build_windows.cmd`
- `README_CMake_flags.md`

## Constraints

- Keep this PR limited to the OpenMP leak.
- Do not mix in ONNX fixes here.
- Do not disable OpenMP globally as the solution.
- Do not use `-forward-slash-prefix-opts` as the fix.
- Do not leave OpenMP in directory-scoped CMake state.

## Implementation checklist

- [x] Reproduce the current failure once and save a verbose Windows build log.
- [x] Confirm at least one CUDA compile line, such as `activation_kernels.cu`, contains both `/openmp:experimental` and `DARKNET_OPENMP`.
- [x] In `CM_dependencies.cmake`, change OpenMP discovery to be host-language-specific. Prefer:
  - `find_package(OpenMP COMPONENTS C CXX QUIET)`
- [x] In `CM_dependencies.cmake`, remove directory-scoped OpenMP mutation:
  - `ADD_COMPILE_DEFINITIONS (DARKNET_OPENMP)`
  - `ADD_COMPILE_OPTIONS (/openmp:experimental)`
  - Appending OpenMP imported targets to the global `DARKNET_LINK_LIBS`
- [x] Replace the above with a simpler internal signal such as `DARKNET_USE_OPENMP` or equivalent boolean state.
- [x] In `src-lib/CMakeLists.txt`, apply OpenMP compile definitions and compile options at the target level, not the directory level.
- [x] Ensure host-only OpenMP state is applied only to non-CUDA compilation.
- [x] Ensure `.cu` files do **not** see `DARKNET_OPENMP`.
- [x] Ensure `.cu` files do **not** see `/openmp:experimental`.
- [x] Link the OpenMP runtime at the final link target where needed, without reintroducing compile leakage into CUDA sources.
- [x] Check `yolo_v2_class` on Windows and make sure it still gets the host-side OpenMP behavior if OpenMP is enabled.

## Recommended implementation shape

Preferred approach:

1. Keep OpenMP discovery in `CM_dependencies.cmake`, but do not mutate compile state there.
2. Store only a boolean such as `DARKNET_USE_OPENMP=ON` when OpenMP is found and allowed.
3. In `src-lib/CMakeLists.txt`:
   - Apply host-only compile definitions to `darknetobjlib`.
   - Apply host-only compile options to `darknetobjlib`.
   - Link OpenMP imported targets where the final link actually needs them.
4. On Windows, use target-scoped compile options guarded so they do not apply to CUDA language compilation.

If target-scoped generator expressions still leak imported OpenMP compile requirements into `.cu` files because `darknetobjlib` is a mixed-language object library, use the fallback below.

## Fallback if the mixed target still leaks

If `.cu` compile lines still pick up OpenMP after the target-scoped fix:

- [x] Split the object build into host and CUDA object libraries.
- [x] Apply OpenMP compile definitions and compile options only to the host object library.
- [x] Keep CUDA object files free of OpenMP compile state.
- [x] Recompose `darknet` from both object libraries.

This is still acceptable for Task 1 if done narrowly and without dragging in ONNX refactors.

## Verification checklist

- [x] A fresh verbose Windows build no longer shows `/openmp:experimental` on any nvcc compile line.
- [x] A fresh verbose Windows build no longer shows `DARKNET_OPENMP` on any nvcc compile line.
- [x] The old nvcc fatal no longer occurs.
- [x] The build proceeds far enough to expose the next blocker, which is expected to be the ONNX/protobuf issue.
- [x] Host `.cpp` compilation still has OpenMP enabled on Windows when OpenMP is found.
- [x] `yolo_v2_class` still links successfully on Windows.

## Acceptance criteria

This task is accepted only if all of the following are true:

- `.cu` compile lines are clean of `/openmp:experimental`.
- `.cu` compile lines are clean of `DARKNET_OPENMP`.
- The build gets past the previous nvcc fatal.
- No ONNX-related code or defaults were changed in this PR, except for comments or test notes.

## Nice to have

- [ ] Add or update a Windows CI job that fails if any CUDA compile line contains `/openmp:experimental`.
- [ ] Add or update a Windows CI job that fails if any CUDA compile line contains `DARKNET_OPENMP`.
