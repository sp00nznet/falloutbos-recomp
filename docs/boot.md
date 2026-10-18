# Boot: from entry point to main menu

What stood between `xbe_entry_point()` and the main menu, in the order it was
hit, with the evidence for each. Every one of these was found by running, not
by reading. Addresses are retail USA `default.xbe` (built 2003-10-02).

## The image

- 28 sections. Besides `.text`, the XDK libraries (`D3D`, `DSOUND`, `XPP`,
  `XGRPH`) and **15 Bink sections** (`BINK`, `BINK32`, `BINKYUY2` x4, the MMX
  blitters `BINK16MX` .. `BINK32M`, and a writable `BINKDATA`).
- Plain `D3D8`, not `D3D8LTCG`, so D3D is a set of ordinary functions.
- No MSVC RTTI: `tools.rtti` finds none, so there are no vtable seeds.
- `.data` is 5.4 MB virtual against 62 KB raw: the engine keeps its pools as
  zero-filled statics.

## 1. D3D's render-state handlers had no bodies

Symptom: `[ICALL] Failed to resolve VA 0x000A6060`, then `0x000A68B0` and
`0x000A7710`, then the 20 s watchdog.

`0x000A6060` is the first byte of the `D3D` section. All three sit in a
30-entry table at `0x000B39F0`, also inside `D3D`, between two functions. 27
of the 30 handlers have direct callers and were found; these three are reached
only through the table. The disassembler's data-pointer pass skipped
executable sections, so it never saw the table.

Fixed in the toolkit (`fix/disasm-tables-in-code-sections`): code sections are
scanned too, but only words no function body covers. It also found the Bink
MMX blitters, which are picked through pointers the same way.

## 2. The GPU fence

Symptom: right after the shaders load (`d:\*.xvu`, `d:\*.xpu`), a spin with
`edx=0x80001000`, the first contiguous allocation.

`sub_000AB9E0` waits for the GPU to consume the pushbuffer:

```
mov esi, [0xb5e58]     ; the device
mov eax, [esi+0x30]    ;   -> fence the GPU writes as it drains
mov ecx, [eax]
mov eax, [esi+0x2c]    ;   submitted
```

The same layout as Half-Life 2's D3D. `main.c` registers
`xbox_Nv2aMirrorFence(0x000B5E58, 0x2C, 0x30)`.

## 3. An XPP entry the sweep stepped over

Symptom: `[ICALL] Failed to resolve VA 0x000D845C`.

The bytes before it are data, and `0d 00 | 55 8b ec` decodes as
`or eax, 0xec8b5500`, swallowing the prologue. Making the table pass realign
at such targets was tried and rejected: re-decoding until the streams converge
split real functions elsewhere (`0x00013180` lost) and invented entries on
consecutive bytes. It is seeded instead, with `tools.seed_from_log`, in
`config/seed_functions.json`, together with `0x00095408`, the main thread's
real entry, handed over through the thread context pointer.

## 4. DirectSound: divide by zero

Symptom: `[EXCEPTION] code 0xC0000094 in sub_000BE0A9`.

Every `CreateSoundStream` returned `DSERR_INVALIDCALL` (`0x88780032`). The
game ignores the result and calls a method on the NULL stream; low guest memory
reads as zeros, so the voice's channel count is 0 and the volume code divides
by it. The creation failed in the voice allocator `sub_000BBFE8`, called on
`[voice+8]`, the MCPX core pointer, which was NULL.

The APU model had seen no register access at all: its registers are plain
zeroed memory unless `RECOMP_AC97_READY` is set. With it, the title starts the
APU itself (`SECTL=0000000F FECTL=0000100F`), its interrupt is claimed, and
every stream creates. `main.c` sets it.

## 5. The streamer and ReadFileEx

Symptom: silence after `d:\resx\arguments.txt`, the main thread in
`NtWaitForSingleObjectEx(..., INFINITE, alertable)`.

The streamer object (at `[0x0010E160]`) has 10 slots, each with an
`OVERLAPPED` at `+0x38 + i*0x14` whose `hEvent` holds one of 10 events. To read,
`sub_00038FB0` registers the slot, then calls `ReadFileEx` with the completion
routine `sub_00038CD0`. `sub_00038EE0` then walks the slots:

```
busy[i] = 1
WaitForSingleObjectEx(slot.hEvent, INFINITE, TRUE)
if (busy[i] == 0) { *done[i] = 1; done[i] = 0; }   ; the completion clears busy
```

The completion routine must run **inside** that alertable wait, which then
returns `STATUS_USER_APC`. The runtime ran it inline, inside `NtReadFile`,
before `busy` was set to 1, so the slot never completed and the next wait
never ended.

Two toolkit fixes (`fix/io-apc-delivery`):

- `xbox_SetDeferredIoApc(1)` queues the APC per thread and runs it at the next
  alertable user-mode wait. Opt-in, since every other title was brought up on
  inline delivery. `main.c` turns it on.
- `deliver_one_apc` added 12 to `esp` after a stdcall routine had already
  popped its 16 bytes. Callers ending in `leave` never noticed; the wait
  wrapper `sub_000907CB` pops `esi` first and handed its caller a stack
  address, so the slot walk ran off the end of the table.

## 6. Bink's MMX probe

Symptom: `[ICALL] target 0x00000000 is not code` right after `empty.bik`
opens, preceded by `[UNIMPL] pushfd`, `popfd` and `cpuid` at `0x000A4E76`.

`sub_000A4E60` is the EFLAGS.ID + `cpuid` MMX probe. As no-ops, `pushfd; pop
eax` popped the saved `ebx`, and the answer was "no MMX". Bink then has no
blitter, `[0x0065B198]` stays NULL, and the blit loop calls it (`0x000CC408`).

`recomp_manual.c` implements the function natively: the Xbox's Pentium III has
MMX. Note the flag sense: the caller's `0x40000` turns MMX **off**; the
original falls through to the clear only when that bit is set. The toolkit fix
is community PR #186; delete the override once a toolkit with it is pinned.

## 7. The controller: contiguous memory on top of the image

With `RECOMP_USB=1` the OHCI model comes up and the title's ISR claims its
interrupts, but `RECOMP_USB_STATS` showed **0 setup packets**. The register
trace (`RECOMP_USB_TRACE=1`) has the driver seeing the connect on port 1
(`HcRhPortStatus = 00010101`), resetting the port (`00100103`), pointing
`HcControlHeadED` at `0x00102940` and setting ControlListFilled, and the model
reading that ED as all zeros, for ever.

`0x00102940` is a physical address. The driver built the ED in contiguous
memory, and in this runtime the contiguous window is separate storage, so the
model has to guess what a physical address names: a contiguous buffer or the
loaded image. It assumes the image whenever the address falls inside it, since
a real kernel never hands out contiguous memory there. This image runs from
`0x10000` to `0x675AB8`; the allocator fills from page 1 up, so the ED sat
inside `.data` as far as the model could tell. Burnout 3, where the model was
proven, has a small image and never overlapped.

`main.c` claims the window pages that alias the image
(`xbox_ContiguousClaim(XBOX_CONTIG_BASE + g_xbox_image_lo, ...)`) before
anything allocates. After it: 8 setups, the pad configured, its interrupt-IN
endpoint polled ~140 times a second, and `RECOMP_PAD_SCRIPT` presses reach the
menus. Doing the same inside the toolkit for every title would be the faithful
fix, but it moves every contiguous allocation of a big-image title such as
Half-Life 2, so it is a separate change.

## 8. A virtual method only a vtable names

In Carbon, `[ICALL] Failed to resolve VA 0x0002F110`, repeating. It is a C++
method after int3 padding that forwards through `jmp [eax+0x24]`; the table
pass requires a closed body and rejects an indirect tail. Seeded.

## Known issues

- Bink frames show scattered 8x8 block artifacts, in the intro and in the menu
  background. The decode is in recompiled MMX code.
- The level runs at about 5 fps in the software pushbuffer executor.
- The opening dialogue box was captured empty once; it may have been mid
  fade-in (the difficulty labels were, in another capture). Unverified.
