# Task 2: Restore a safe ONNX default on Windows

## Goal

Make the default Windows build skip ONNX unless the user explicitly opts in.

This is a user-facing mitigation task. It is **not** the structural ONNX fix.

## Why this task exists

As of current `master`:

- `CM_dependencies.cmake` defaults `DARKNET_TRY_ONNX` to `True` when not defined.
- `README_CMake_flags.md` documents that V5.0 defaulted ONNX to `OFF`, and V5.1 changed the default to `ON`.
- The supplied Windows logs show that once OpenMP is removed from the way, CUDA compile lines still contain `-DDARKNET_HAS_PROTOBUF` and fail in protobuf headers.

This means Windows users are being pushed into a broken path by default.

## Files to inspect and edit

Required:

- `CM_dependencies.cmake`
- `README_CMake_flags.md`

Possibly relevant:

- `build_windows.cmd`
- `README.md`
- any Windows-specific build documentation

## Constraints

- Keep this PR small.
- Do not attempt the structural ONNX isolation here.
- Do not change explicit opt-in behavior on Windows.
- Avoid changing defaults for non-Windows platforms in this PR unless the maintainer explicitly wants that.

## Implementation checklist

- [x] In `CM_dependencies.cmake`, change the undefined default for `DARKNET_TRY_ONNX` so Windows defaults to `False`.
- [x] Preserve explicit opt-in behavior:
  - `-DDARKNET_TRY_ONNX=ON` must still enable the ONNX path.
- [x] Add a clear configure-time status message for Windows builds explaining that ONNX export is off by default until the protobuf/CUDA isolation work lands.
- [x] Keep the existing configure-time message when ONNX is explicitly skipped.
- [x] Update `README_CMake_flags.md` to document the Windows-specific default.
- [x] If there is Windows build documentation, add a short note showing how to opt in explicitly:
  - `cmake -DDARKNET_TRY_ONNX=ON ...`

## Recommended implementation shape

Preferred behavior:

- If `DARKNET_TRY_ONNX` is not defined:
  - `WIN32` -> default to `False`
  - non-Windows -> keep current behavior for now

Keep the policy change tightly scoped so it can be reviewed and merged quickly.

## Verification checklist

- [x] A default Windows configure no longer enables ONNX automatically.
- [x] A default Windows configure emits a clear message that ONNX is skipped by default on Windows.
- [x] A default Windows verbose build no longer shows:
  - `DARKNET_HAS_PROTOBUF` on CUDA compile lines
  - `src-onnx` include paths on CUDA compile lines
- [x] The default Windows build does not attempt to build `darknet_generated_proto`.
- [x] Explicit `-DDARKNET_TRY_ONNX=ON` still enters the ONNX path.

## Acceptance criteria

This task is accepted only if all of the following are true:

- A default Windows build skips ONNX automatically.
- The default Windows build no longer drags protobuf/ONNX state into CUDA compilation.
- The opt-in switch still works.
- No structural ONNX refactor was mixed into this PR.

## Nice to have

- [ ] Add or update a Windows CI job for the default configuration with no explicit ONNX flag.
- [ ] Add a non-blocking Windows CI job for `-DDARKNET_TRY_ONNX=ON` so the remaining work stays visible.
