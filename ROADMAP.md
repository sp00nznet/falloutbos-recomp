# Roadmap

## Next
- Speed in the level: ~15 fps after #228. The title's thread still spends its
  time waiting for the executor in D3D's kickoff; next is the executor's own
  per-batch work (vertex programs, combiner setup), or a hardware back end.
- Play past the opening dialogue with a real pad; confirm dialogue text draws.
- Upstream the image-aliasing claim into xboxrecomp's contiguous allocator.
- Bink block artifacts: find the recompiled MMX routine that decodes them
  wrong (compare one frame against xemu as a black box).
- Get the three toolkit changes merged and pin the toolkit version; then delete
  the `sub_000A4E60` override.

## Required by REPO_RULES, not done yet
- `Setup.cmd` Quick start (section 5): unpack the disc, regen, build, shortcut.
- `--headless --record out.mp4` (section 10): xboxrecomp has no ffmpeg pipe
  yet. Today: `RECOMP_FB_DUMP` frames, which work headless.
- Conformance harness (section 9): a fixed boot script that reports how far the
  title got (logo, intro, menu) as a pass/fail count.

## Later
- Start New Game: load the first level.
- Save games on the HDD partition (`UDATA\49500008`).

## Out of scope
- Distributing game data or generated C, in any form.
- The PS2 version.
