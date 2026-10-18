# Roadmap

## Next
- Speed in the level: ~15 fps after #228. The title's thread still spends its
  time waiting for the executor in D3D's kickoff; next is the executor's own
  per-batch work (vertex programs, combiner setup), or a hardware back end.
- A camera fault seen once in Carbon (docs/images/known-camera-wedge.png): a
  giant wedge fills the screen while the minimap is unchanged. Not reproduced
  since; catch it with a frame trace.
- Upstream the image-aliasing claim into xboxrecomp's contiguous allocator.
- Bink block artifacts. Ruled out: threading (same with RECOMP_GUEST_LOCK=1)
  and the MMX helpers (tests/mmx_integer passes). Next: decode one .bik frame
  with ffmpeg as the reference and diff the decoder's output buffer.
- Get the three toolkit changes merged and pin the toolkit version; then delete
  the `sub_000A4E60` override.

## Required by REPO_RULES, not done yet
- `--headless --record out.mp4` (section 10): xboxrecomp has no ffmpeg pipe
  yet. Today: `RECOMP_FB_DUMP` frames, which work headless.

## Later
- Start New Game: load the first level.
- Save games on the HDD partition (`UDATA\49500008`).

## Out of scope
- Distributing game data or generated C, in any form.
- The PS2 version.
