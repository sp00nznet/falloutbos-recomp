# Roadmap

## Next
- Controller input: bring up the USB/XID path the title's XAPI expects, so the
  menu can be driven. hl2-recomp `docs/input.md` is where that stands.
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
