# AArch64 GX Investigation Notes

Date: 2026-09-04

Context:
- `x86_64-linux` renders Mario Kart Wii correctly.
- `aarch64-linux` renders with severe geometry corruption, most visibly on racer and character meshes.
- The issue was reproduced on Asahi Linux hardware.

Observed behavior:
- Track geometry, UI, and some simpler assets improved after forcing a CPU-expanded indexed-vertex fallback.
- Racer meshes remained visibly corrupted after that change.
- The corruption pattern remained stable across several shader-side experiments, which argues against this being a simple texture or lighting issue.

Artifacts:
- `2026-09-04 09-44-30.png`: earlier, more severe corruption.
- `2026-09-04 12-10-12.png`: improved track/UI state, but racers still broken.

What appears to help:
- `aurora-main/lib/gx/command_processor.cpp`
  - Added an `aarch64-linux` CPU-expanded indexed-vertex fallback for raw draw submission paths.
  - This improved some indexed geometry and some UI-related assets.

What was added mainly for diagnosis:
- `runtime/src/hle/gx/gx_dl.cpp`
  - Added `aarch64-linux` display-list fast-path bypass when indexed attrs or indexed XF arrays are present.
  - Added low-volume logs confirming that bypass is actually being exercised.
- `runtime/src/hle/gx/gx_dl.cpp`
  - Added a CPU-resolved indexed-XF packet path for the display-list interpreter.

Changes that are probably correct, but did not resolve the racer corruption:
- `aurora-main/lib/gx/gx.hpp`
- `aurora-main/lib/gx/gx.cpp`
- `aurora-main/lib/gx/command_processor.cpp`
  - Indexed array cache generation tracking and direct-indexed fallback plumbing.
- `runtime/src/hle/gx/gx_vertex.cpp`
  - Forward matrix-index descriptors to native GX as `GX_DIRECT` when enabled.
- `runtime/src/hle/gx/gx_fifo.cpp`
- `runtime/src/hle/gx/gx_dl.cpp`
  - Replay CP matrix-index regs `0x30` and `0x40` to native GX on the fallback paths.

Experiments that did not produce the fix:
- `aurora-main/lib/gx/shader.cpp`
  - WGSL matrix-row rewrite. No visible fix.
- Additional shader-side fetch/attribute experiments.
  - Some caused different lighting/color behavior but did not resolve the racer geometry issue.
  - One broader fetch experiment made rendering significantly worse and was reverted before this note was written.

Working conclusion:
- The current patch stack is not an effective fix for the `aarch64-linux` racer/character corruption.
- The CPU-expanded indexed-vertex fallback is a partial mitigation only.
- The remaining likely problem is deeper in Aurora's matrixed/skinned draw handling, or in a submission path that still differs for these character meshes.

Recommended next step:
1. Add targeted logs around the actual bad character draw path.
2. Confirm which submission path those draws take:
   - display-list interpreter
   - raw draw fallback
   - immediate `GXBegin` path
3. Log live matrix state at submission:
   - `PNMTXIDX`
   - current matrix index
   - whether the draw is using per-vertex matrix indices or current-matrix state only
4. Only then attempt the next behavioral patch.
