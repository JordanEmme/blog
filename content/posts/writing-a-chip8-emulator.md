+++
title = 'Writing a CHIP-8 Emulator'
date = '2024-10-24T15:34:57+02:00'
draft = true
toc = true
tocBorder = true
[params]
  author = "Jordan Emme"
+++

Where I document how I started learning about emulation and wrote my first toy
model.

## Why?

I have spent a lot more time than I would care to admit playing old SNES
(amongst others) games when I was young and carefree. Being able to experience
Secret of Mana and the likes on a computer always seemed like magic to me, but
it had never occured to me then that I could just learn the trick behind the
illusion. So a few weeks ago, I decided to start learning doing just that, by
writing my own CHIP-8 emulator.

I have had tons of fun doing it, and hope I can convey that excitement in
this post. Additionally, if you are also interested in learning how to write
emulators, this is a very manageable project to scratch that itch. Should you
want to tackle it, I hope you find pretty much all the info you need here, even
though this is not written as a tutorial.

## The CHIP-8

### What Is It?

The CHIP-8 is an interpreted language created by Joseph Weisbecker, to run on
a 1802 microprocessor. It provides a simple and lightweight language to code
with on these older chips. Whilst not being an actual, physical machine, writing
an interpreter (not emulator) for the CHIP-8 is seen as the "Hello World!" of
emulation. Indeed, the CHIP-8 is designed as a virtual machine, which comes with
its own display, memory layout, CPU architecture and instructions. As such, let
us not be pedantic and delve into how to write a CHIP-8 __emulator__ on a modern
machine.

### Specifications Overview

We give a few words on the CHIP-8 specs, but will dive into more details in the
next section.

#### "Hardware"

The CHIP-8 consists of:
* A CPU that typically runs at 700Hz (although implementations may vary);
* 16 registers (15 general purpose variables + 1 flag);
* A program counter;
* An index register to point at locations in memory;
* A stack (usually 32 bytes but may vary);
* A whopping 4096 bytes of memory;
* A hexadecimal keypad;
* An 8-bits delay timer;
* An 8-bits sound timer;
* A 60Hz, 64x32, monochrome display.

That is all the "hardware" that we have to emulate. This can be done in a very
straightforward many, as this post will, hopefully, illustrate.

#### Font

On top of that hardware, the CHIP-8 should have a font loaded in memory. Here is
a basic font that most people seem to be using:

```
0xF0, 0x90, 0x90, 0x90, 0xF0, // 0
0x20, 0x60, 0x20, 0x20, 0x70, // 1
0xF0, 0x10, 0xF0, 0x80, 0xF0, // 2
0xF0, 0x10, 0xF0, 0x10, 0xF0, // 3
0x90, 0x90, 0xF0, 0x10, 0x10, // 4
0xF0, 0x80, 0xF0, 0x10, 0xF0, // 5
0xF0, 0x80, 0xF0, 0x90, 0xF0, // 6
0xF0, 0x10, 0x20, 0x40, 0x40, // 7
0xF0, 0x90, 0xF0, 0x90, 0xF0, // 8
0xF0, 0x90, 0xF0, 0x10, 0xF0, // 9
0xF0, 0x90, 0xF0, 0x90, 0x90, // A
0xE0, 0x90, 0xE0, 0x90, 0xE0, // B
0xF0, 0x80, 0x80, 0x80, 0xF0, // C
0xE0, 0x90, 0x90, 0x90, 0xE0, // D
0xF0, 0x80, 0xF0, 0x80, 0xF0, // E
0xF0, 0x80, 0xF0, 0x80, 0x80  // F
```

These numbers represent sprites on the CHIP-8's monochrome display. When writing
the five numbers that make up 0 in binary, one gets:

```
11110000
10010000
10010000
10010000
11110000
```

It should be clear from that example that, with digit 1 representing an
"on" pixel, and 0 an "off" pixel, 5 bytes represent an 8x5 pixel sprite on a
monochrome display.

## How to Emulate

Let's get into the nitty-gritty and look at what we need to do to actually
emulate the CHIP-8.

### Prerequisite

Although we are technically writing a CHIP-8 interpreter, it is clear from the
specs that this thing comes with a display, a keypad, and some sort of sound
capacity. So if we want to see, and hear, our emulator come to life, we need to
be able to render that display, give inputs, and play some sound (just a buzzer
really...).

I used SDL2 for all of that, which, though simple, is still pretty
overkill for a project that size. That said, I was not keen on creating and
handling a Wayland surface and making window decorations (thanks Gnome), let
alone diving in the hellscape that is playing sounds from scratch. Remember
kids, self-harm is never the answer.

### Data Needed

We need to represent all the hardware specs detailed in the [Specifications Overview](#specifications-overview) section with some data. So, we should see these different variables somewhere in our code:

```cpp
  uint8_t memory[4096];
  bool display[64 * 32];
  bool keypad[16];
  uint16_t stack[16];
  uint8_t V[16];          // 16 registers
  uint16_t pc;            // Program counter
  uint16_t I;             // Memory index
  uint16_t sp;            // Stack pointer
  uint8_t soundTimer;
  uint8_t delayTimer;
 ```

> Remarks:
>
> 1. We are using a dedicated array for the stack. We could (and, if we wanted
to remain more faithful to how the CHIP-8 is supposed to work, should) put the
stack somewhere in memory instead, as there is enough dedicated space in it for
such purposes. We chose not to, out of laziness.
> 2. Instead of using a 64x32 array of bools to store the display state, we
could use an array of 64 bits (unsigned) integers of size 32, if we were more
conservative about memory. We chose not to, for the same reason as in 1.
> 3. Similar remark for the keypad which could just be a single 16 bit integer
where each bit represents whether a key is up or down. In my case, I didn't
use either of these, and let SDL2 do the job of updating my keyboard states and
grabed a handle to that data.

That is pretty much the data we need to play with in order to get this thing off
the ground.

### Fetch-Decode-Execute Cycle

OK, so now we've got all the hardware sorted but what do we actually do with it?

### The Display

### Timers

### Clock Speed

## Further Improvements

## Testing

## References
