---
layout: post
title:  "Animating Sleep Zs on the Neo6502 (and how the firmware ate my input)"
date:   2026-05-01 19:00:00 +0100
categories: retrocomputing assembly neo6502 commanderx16
---

I've been putting some polish into [From the Dead](https://github.com/andymccall/from-the-dead), the post-apocalyptic action-RPG I've been building in pure assembly for the Commander X16, Neo6502 and Agon Light 2.  One of the smaller pieces in the v1.2 milestone is a sleep cutscene: walk up to the bed in the indoor scene, pick "Yes" on the "Go to sleep?" prompt, and the room fades to black while a `... S N O R E !!! ...` dialogue prints out and some animated Zs fade in and out across the screen.  Press A and you fade back in to find the player has been repositioned to the side of the bed.  Small bit of mood-setting before I get to the meatier story stuff.

The Commander X16 version went together first and came out exactly the way I wanted:

<video src="/assets/images/20260501_commander_x16_sleep.webm" controls width="640">
Your browser does not support the video tag.
</video>

Four Zs at a time, each fading through six phases (dark grey, light grey, white, light grey, dark grey, then black to erase), scattered across the centre 60% of the screen with a bit of LFSR-driven jitter so they don't form patterns. Letter `Z` and `z` mixed randomly so they don't all look identical.  The snore types out, the return-key glyph latches when it's done, A on the SNES controller dismisses, room fades back in, lovely.

Then I ported it to the Neo6502.

## The first sign that something was wrong

Same logic, same state machine, same constants.  On Neo the only thing that should be different is the renderer, because Neo has a different graphics path - more on that below.  But the first run produced this:

![Neo6502 sleep cutscene with the snore text stuck at E](/assets/images/20260501_neo6502_snore_stuck_at_e.png)

The snore text types `... S N O R E` and then stops.  The Zs are scattering away in the upper portion of the screen, looking great.  But the `!!! ...` never appears, and pressing keys on the keyboard does nothing.  I'm just sat there pressing buttons at a screen that won't move on.

If I let it sit, sometimes I'd eventually get the return-key glyph to appear (so the typer's state machine *had* completed - it just wasn't drawing the trailing chars visually).  And sometimes pressing keys on the keyboard would type the literal characters into the bottom of the dialogue box in black, like the program had crashed back to a firmware console:

![Neo6502 with firmware crash text stamped over the dialogue box](/assets/images/20260501_neo6502_firmware_crash.png)

Two distinct symptoms:

1. The snore typer stops drawing somewhere around char 13 (`E`)
2. A press doesn't register, so the dialogue can't be dismissed

Two symptoms, probably the same root cause.

## The investigation

I had a guess that it was about CPU saturation - the X16 typer redraws the whole prefix on each reveal (so by char 20 it's painting ~600 pixels in one frame) and Neo's per-pixel firmware API is much slower than X16's direct VRAM writes.  But guessing isn't science.  So I started chopping things out one at a time.

The bisect was something like:

1. **Disable the Z animation entirely.**  Snore types out fully, A dismisses.  ✓ It's the Zs.
2. **Keep the Z state machine, stub out `draw_zs_at_x` (no actual drawing).**  Still works.  So it's specifically the *drawing* in `draw_zs_at_x`, not the state ticking.
3. **Drop `SLEEP_ZS_COUNT` from 4 to 1, keep full Z drawing.**  Still broken.  So it's not pure CPU saturation from multiple Zs - even one is enough.
4. **Keep `set_graphics_color` calls but skip `TEXT_DrawString`.**  Works.  So it's `TEXT_DrawString` (the per-pixel glyph draw), not the colour change.
5. **Replace the `Z` glyph with a space character (`TEXT_DrawString` runs through its iteration logic but emits no actual `DRAW_PIXEL` calls).**  Works.  So it's specifically the pixel writes, not the iteration.
6. **One single raw `DRAW_PIXEL` per Z phase advance.**  Works fine - the snore completes, A dismisses, the single pixel fades up and down in the corner.
7. **Full Z glyph (~22 `DRAW_PIXEL` calls per phase advance).**  Broken again!

So the threshold sits somewhere between 1 and 22 firmware API calls per phase advance.  Below it, everything works.  Above it, the typer stops drawing and the keyboard stops responding - **at the same time**, which strongly hinted both symptoms had the same root cause.

## What's actually happening

The Neo6502's graphics API is a "mailbox" at `$FF00..`.  To draw a single pixel I write x, y, function ID, and a group/trigger byte; the firmware notices the trigger, processes the command, and zeroes the trigger byte to signal done.  Every pixel.  Every. Single. Pixel.

The firmware can only do one thing at a time.  And one of the *other* things it has to do is poll the keyboard - notice when buttons go from up to down, update its internal state, expose that to my code via another API call.  It does this in the gaps between API requests.

When `TEXT_DrawString` fires ~22 `DRAW_PIXEL` calls back-to-back in a tight loop, with each iteration only waiting just long enough for the firmware to zero the trigger byte, there are no gaps.  The firmware is so busy answering pixel requests that it can't get round to polling the keyboard.  When I press a key, the bit goes high on the physical (well, emulated at this stage!) hardware - but by the time the firmware actually reads the keyboard again, the bit has already flipped back to 0.  So my game code's `INPUT_PollOnce` never sees the rising edge.

The "snore stops at E" symptom is the same effect: the typer's `draw_partial` redraws the whole prefix on every reveal, peaking at ~600 pixel writes in a single frame on the last reveal, and the firmware ends up dropping pixel writes (or the next setup overwrites the in-flight one, I'm not sure exactly which from outside the firmware) once the queue gets backed up.

The black-text-over-the-blue-box screenshot from earlier is the same thing taken to its conclusion - the program has lost track of state badly enough that the firmware has dropped back to its default console mode and is echoing my keypresses straight to the screen.

## Why the X16 doesn't have this problem

The Commander X16 has the VERA video chip mapped directly into 6502 memory.  To draw a pixel, my code just stores a byte to a memory address - no firmware involvement, no mailbox protocol, no input poll competing for cycles.  600 "pixel writes" on X16 is just 600 `STA` instructions that complete deterministically and don't block anything else.  That's why the same logic that breaks the Neo runs completely happily on the X16.

I knew the architectures were different going in, of course, but I didn't expect it to bite quite this hard quite this early.  Lesson learned.

## The fix (well, the *current* fix)

Once I understood the problem the path forward was obvious: stop using `TEXT_DrawString` for the Zs, because per-pixel drawing is exactly what eats the firmware budget.  Use `DRAW_RECT` instead - one API call paints a whole rectangle, with the same internal pixel work but only *one* round-trip through the mailbox.

I broke the Z down into four rectangles:

```
######      <- top bar, one DRAW_RECT
     #      <- single-pixel diagonal step
    #
   #
  #
######      <- bottom bar, one DRAW_RECT
```

That's six `DRAW_RECT` calls per Z phase advance instead of ~22 `DRAW_PIXEL` calls.  Worst case - all four Zs phase-advance in the same frame - is 24 firmware API calls, plus whatever the typer's still doing on its peak frame, total around 140 round-trips.  Comfortably under the threshold I'd been crashing into.

After landing this, the snore types out cleanly, the A press registers and dismisses, and the Zs scatter properly through the centre 60% with a smooth fade.  Not pixel-identical to the X16 (those are real letter `Z` and `z` glyphs at the font's 8×8 size, mine are stylised 6×6 stepped Zs), but close enough that the cutscene reads close to the way I wanted:

<video src="/assets/images/20260501_neo6502_sleep.webm" controls width="640">
Your browser does not support the video tag.
</video>

There's also a lesson here for when I get to the Agon Light 2 port: it's another firmware-mediated graphics path, so I'd expect similar dynamics.  Prefer the chunky one-call primitives like `DRAW_RECT` over per-pixel work, and keep an eye on the input-poll budget.

## What's next

There's still a path to true X16 parity if I want it.  Neo has an `API_FN_DRAW_IMAGE` function (`$07` in the graphics group) - the same single-call image blitter that paints the room tiles.  I could pre-render the Z and z glyphs (and maybe a couple of size variants) into the image pool and stamp them with one API call each.  That would let me have the literal `Z` and `z` letterforms back, at any size, with even *less* firmware pressure than the current `DRAW_RECT` approach.  The wrinkle is colour fades - `DRAW_IMAGE` blits baked palette indices, so I'd need either pre-coloured variants or some palette mutation trickery to get the six-phase fade.

I'm going to try that next - it's the closest I can get to dropping the Neo's sleep cutscene right next to the X16 video at the top of this post and not being able to tell them apart.  I'll update this post with the result once I've got it landed.

In the meantime, if you've ever wondered why "just draw a few sparkles" on a firmware-API system can punch through your input handling - now you know.
