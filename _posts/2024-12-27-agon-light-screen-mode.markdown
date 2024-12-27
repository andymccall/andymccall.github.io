---
layout: post
title:  "Saving the MOS Screen Mode in Assembler"
date:   2024-12-27 17:14:47 +0100
categories: electronics retrocomputing agon basic
---

I've been working on my assembly language skills in the last few weeks and wrote a [little application](https://github.com/andymccall/bitriotdev) to try various things on the Agon Light 2, such as displaying shapes and playing sounds.  Richard Turnnidge, who's got some excellent [YouTube videos](https://www.youtube.com/watch?v=NFgZcnyV8mU) and [assembly lessons](https://github.com/richardturnnidge/lessons), tried my application and pointed out to me  on the Discord channel that his original screen resolution wasn't restored after the application quit. I added that back to my code.

Here's a clear example of how to do it.  You grab the current mode from the system variables, store the mode in memory and just before quitting set the mode back to whatever was stored.

```
.assume adl=1
.org $040000

    jp start

; MOS header
.align 64
.db "MOS",0,1
 
; API includes - you may need to adjust the path
    include "mos_api.inc"
 
    start
 
; Push the stack
    push af
    push bc
    push de
    push ix
    push iy
 
; Get the current screen mode

    ; Make a call to get the system variables
    MOSCALL mos_sysvars
    ; Get the current screen mode
    ld  a, (IX + sysvar_scrMode)
    ; Save the current screen mode to memory
    ld (previous_screen_mode), a
 
    ; Setting the screen mode - https://agonconsole8.github.io/agon-docs/vdp/Screen-Modes/#vdu-22-mode
    ; Screen modes - https://agonconsole8.github.io/agon-docs/vdp/Screen-Modes/#screen-modes_1

; Change the screen mode
    ; Tell the VDP we're going to be setting the screen mode 
    ld a, 22
    ; Send it to the VDP
    rst.lil $10
    ; Put 8 into the accumulator, which is 320x240x64
    ld a, 8
    ; Send it to the VDP, settings the screen mode
    rst.lil $10

; Resore the previous screen mode

    ; Tell the VDP we're going to be setting the screen mode
    ld a, 22
    rst.lil $10
    ; Move previous_screen_mode to the accumulator
    ld a, (previous_screen_mode)
    ; Send it to the VDP, settings the screen mode
    rst.lil $10
 
; Pop the stack
    pop iy
    pop ix
    pop de
    pop bc
    pop af
    ld hl,0
 
    ret

; Variable to store the current screen mode
previous_screen_mode:
    .db 0
```

Hopefully this will help someone in the future.