+++
title = 'Writing a CHIP-8 Emulator'
date = '2024-10-24T15:34:57+02:00'
draft = false
toc = true
tocBorder = true
math = true
[params]
  author = "Jordan Emme"
+++

![chip](chip8.png)

Where I document how I started learning about emulation and wrote my first toy
model.

## Why?

I have spent a lot more time than I would care to admit playing old SNES
(amongst others) games when I was young and carefree. Being able to experience
Secret of Mana and the likes on a computer always seemed like magic to me, but
it had never occurred to me then that I could just learn the trick behind the
illusion. So a few weeks ago, I decided to start learning doing just that, by
writing my own CHIP-8 emulator.

I have had tons of fun doing it, and hope I can convey that excitement in
this post. Additionally, if you are also interested in learning how to write
emulators, this is a very manageable project to scratch that itch. Should you
want to tackle it, I hope you find pretty much all the info and references you
need here to get you most or all of the way there.

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

---
{data-content = "\"Hardware\""}

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
* A 60Hz, 64x32, monochrome display;
* A 16 digits keypad.

That is all the "hardware" that we have to emulate. This can be done in a very
straightforward many, as this post will, hopefully, illustrate.

---
{data-content = "Font"}

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

---
{data-content = "Keypad"}

The CHIP-8 keypad is made up of 16 digits (hex numbers between 0 and F), with the layout usually being like so:

|  1  |  2  |  3  |  C  |  
|:---:|:---:|:---:|:---:|
|**4**|**5**|**6**|**D**|
|**7**|**8**|**9**|**E**|
|**A**|**0**|**B**|**F**|

It's our choice entirely how we want to assign each key, but the usual choice seems to be to use the left portion of the (QWERTY) keyboard and have that keypad map to:

|  1  |  2  |  3  |  4  |  
|:---:|:---:|:---:|:---:|
|**Q**|**W**|**E**|**R**|
|**A**|**S**|**D**|**F**|
|**Z**|**X**|**C**|**V**|

Note that depending on your implementation, you may need mappings in both
directions.

## How to Emulate

Let's get into the nitty-gritty and look at what we need to do to actually
emulate the CHIP-8.

### Prerequisites

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

We need to represent all the hardware specs detailed in the [Specifications
Overview](#specifications-overview) section with some data. So, we should see
these different variables somewhere in our code:

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
could use an array 32-bits (unsigned) integers of size 64, if we were more
conservative about memory. We chose not to, for the same reason as in 1.
> 3. Similar remark for the keypad which could just be a single 16 bit integer
where each bit represents whether a key is up or down. In my case, I didn't
use either of these, and let SDL2 do the job of updating my keyboard states and
grabbed a handle to that data.

That is pretty much the data we need to play with in order to get this thing off
the ground.

### Loading Font and ROM Data

Before we start the actual emulation cycle, we should initialise some things. On
top of setting up a window to render in, and something to play sound, the CHIP-8
requires some data to be loaded in RAM.

As mentioned in the [Specifications Overview](#specifications-overview), the
CHIP-8 requires a font to be loaded in memory. Any location between `0x000` and
`0x1FF` is fair game, as this part of the RAM is dedicated to the interpreter/
emulator we are building. Should we want to have the stack emulated as part of
the RAM, this is also where we would put it. I, somewhat arbitrarily, decided to
copy the font data at the start of the RAM, at address `0x000`.

Furthermore, since we obviously want to test our emulator on some real CHIP-8
programs, we have to also load it in memory. The specifications I found
regarding the CHIP-8 RAM layout indicated that a ROM should typically be loaded
after memory address `0x200`.

At this stage, our `main` function looks something like:

```cpp
int main(int arc, char** argv){
  setup_window(); // where we initialise SDL, or whatever we use
  load_font();
  load_rom();

  return 0;
}  
```

### Fetch-Decode-Execute Cycle

This is where the magic actually happens, and what the meat of the program
actually does. In order for each instruction of the program to be executed,
three things need to happen. First, we need to grab the instruction's opcode at
the address given by the program counter `pc`. This is the *fetch* step.

Then, we get a 2-byte opcode, which can be represented by a number of
length 4 in hexadecimal. We need to *decode* what instruction it matches to
(more words on that later).

Finally, we can *execute* that operation, *i.e.* actually change the states of
the CHIP-8 accordingly.


Let us dive into more details on how to make this happen.

---
{data-content = "Fetch"}

This is the easy part. We just need to grab the two consecutive bytes in memory
at the address given by the program counter, since an opcode is a 16-bit number
(or, a 2-bytes number).

The CHIP-8 is a big-endian system, so the byte stored at address `pc` is
the most-significant one, and the one stored at address `pc + 1` is the
least-significant one. The `fetch` function should therefore look something
like this:

```cpp
uint16_t fetch(){
  uint16_t opcode = (memory[pc] << 8) | memory[pc + 1];
  pc += 2; // we increment the program counter
  return opcode;
}
```
---
{data-content = "Decode and Execute"}

We have now gotten hold of the opcode, it is time to decode it. A complete
list of supported operations and their code can be found [here](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM#3.1).
Nevertheless, let us say a few words about the general structure of the opcode.

Chip-8 opcodes come in, roughly speaking, three flavours. Writing the 16-bit
opcode as four hex digits, it can take one of three forms:

1. `0xHxyn`
1. `0xHxkk`
1. `0xHnnn`

where
* `H` is an hexadecimal digit which partially encodes the type of operation to
be done;
* `n` is a 4-bits nibble, either specifying the operation further, or used as an
operation argument;
* `kk `is a byte, often used as an operation argument;
* `nnn` is a 12-bits nibble, which always encodes for a memory address;
* `x` is a 4-bits integer, always specifying the register to be accessed for the
operation;
* `y` is a 4-bits integer, always specifying the register to be accessed for the
operation.

Given the overall simplicity of most of the operations, I have decided to wrap
both the decoding and executing phases in a single function, which is basically
a big switch case on values of `H`. It basically looks like:

```cpp
void decode_and_execute(uint16_t opcode){

  // First we define all the nibbles to avoid code duplication
  uint8_t x = (opcode >> 8) & 0xF;
  uint8_t y = (opcode >> 4) & 0xF;}
  uint8_t n = opcode & 0xF;
  uint8_t kk = opcode & 0xFF;
  uint16_t nnn = opcode & 0xFFF;

  switch (opcode & 0xF000){
    case 0x0000:
      // Execute 0nnn, or 00E0, or 00EE accordingly.
      break;
    case 0x1000:
      // Jump to memory location nnn
      break;
    case 0x2000:
      // Call subroutine at nnn
      break;
    .
    .
    .
  }
```

We will not detail any of the implementations here, but most of the operations
are fairly simple, with a few notable exceptions (such as `0xDxyn` which draws a
sprite of size 8 by `n` at coordinates `(x, y)`). We will insist on this in the
[display section](#the-display), as well as [some pitfalls](#some-pitfalls).

---
{data-content = "General Structure"}

Now, assuming we do the decoding and execution steps in a single function, our
`main` function should now look something like this:

```cpp
int main(int argc, char** argv){
  setup_window(); // where we initialise SDL, or whatever we use
  load_font();
  load_rom();

  bool running = true;
  uint16_t opcode;
  while(running){
    opcode = fetch();
    decode_and_execute(opcode);
    running = check_if_quit();
  }
  close_window();

  return 0;
}
```
### The Display

As mentioned before, the display is a `64x32`, monochrome display. There are
only two instructions which change the state of the display:
1. `0x00E0`, which clears the display, so we just need to set all the pixels
to "off";
1. `0xDxyn`, which draws a sprite of size `8xn` at coordinate `(x,y)` on the
screen. The sprite is stored in the `n` bytes at `I` in memory.

---
{data-content = "The 0xDxyn Instruction"}

Let us give some more details about the `0xDxyn` instruction. First of all,
to remove any and all ambiguity regarding the position of the sprite, the
coordinate `(x, y)` refers to the top-left corner of the sprite.

Secondly, we read `n` bytes from memory, between offsets `I` and `I + n - 1`.
Each of these bytes represents a row of 8 pixels in our sprite. These pixels
then get XORed with the existing display pixel states. One can think of the
sprite 0s and 1s as, respectively, "don\'t change that pixel's state" and "flip
that pixel's state"

Finally, we need to consider some possible edge case, namely where the
sprite goes "off screen" (if `x + 8 >= 64` or `y + n >= 32`). I have found
contradicting specs regarding that issue, some claiming that the sprite then
needed to wrap around the display, (so basically doing things mod 64 in the
x direction and, sometimes but not always, mod 32 in the y direction), and
some claiming that the sprite should be culled should it go off the screen
boundaries. A fully-featured emulator should implement all versions with a
flag and leave the option to the user, as different programs will operate on
different assumptions with the drawing operation behaviour.

---
{data-content = "Displaying the Pixel Data"}

In my case, I found it easiest to separate the notion of "display", which
is just an array of booleans representing the pixel states, and the actual
rendering in the app window. It makes it extremely simple to refresh the display
at set intervals (specs dictate 60Hz) by just calling a method which takes these
pixel states and renders an appropriate image.

That sort of decoupling is also necessary since you do not want to refresh the
display every time this instruction is called. Indeed, the CHIP-8 display has a
refresh rate of 60Hz whereas the CHIP-8 typically runs at 700Hz. We will touch a
bit more on that later.

### Timers

As written in the [Specifications Overview](#specifications-overview) section,
the CHIP-8 also has two timers. A general purpose delay timer, which can
basically act as a timestamp for the CHIP-8, and a sound one, which controls
when a sound should be played.

Both timers are 8-bits long unsigned integers. They get decremented (if they are
non zero) at a 60Hz rate. Furthermore, a sound is played if the sound timer is
greater than zero.

### Clock Speed

A final word that pertains on the general structure of our main loop, is that
the CHIP-8 should generally run at 700Hz, and the display be updated at 60Hz,
which means that one should run several fetch-decode-execute cycles before
refreshing the screen.

A quick hack I chose to implement for an approximation of that behaviour
was to grab my display refresh rate through SDL, which is about 60Hz, force
vertical synchronisation of the display, and run $\left\lfloor\frac{700}
{\text{refresh rate}}\right\rfloor$ fetch-decode-execute cycles before I update the
display. Obviously, if one would like to run it on a display with a higher
refresh rate, then one should implement a timer to ensure we still only render
at 60 frames per second.

After these considerations, the main function essentially resembles this:

```cpp
int main(int argc, char** argv) {
  setup_window();
  load_rom();
  load_font();

  bool running = true;
  uint16_t opcode;
  uint8_t cyclePerFrame = CLOCK_SPEED / 60;
  while (running){
    for (uint8_t i = 0; i < cyclePerFrame; i++) {
      opcode = fetch();
      decode_and_execute(opcode);
    }
    update_timers();
    refresh_display(); // should be called at 60Hz, either through timers/ticks or VSync
    running = check_if_quit();
  }
  return 0;
}
```

## Some Pitfalls

In no particular order, here is a list of the silly mistakes I made. These are
hopefully common enough that they can help you debug strange behaviours of the
emulator:

1. In some of the relevant instructions, using `x` in lieu of `V[x]`;
1. In some of the relevant instructions, changing the flag register value `VF`
before executing main bit of the instruction on the other registers.
1. In the `0xDxyn` instruction, reading the sprite data bits from lowest to
highest resulting in applying pixel data right-to-left instead of left-to-right,
thus mirroring sprites.

## Further Improvements

Some things that I don't want to spend time on, but would be necessary to have a
more fully-fledged emulator, that is compatible with more programs would be:

1. Having a better solution for the clock speed, and implementing variable
clock-speed which can be defined by the user (or better yet, changed at
runtime.);
1. Some instructions, such as the one coded `0x8xy6` for instance, can have
different implementations. In some implementations, this sets `Vx` to `Vx &
1`,and in some others, it sets it to `Vy & 1` (which seems more sensible seeing
as we have access to `y` from the opcode). Having a flag to switch between
implementations is necessary;
1. Similar remark for the behaviour of sprites going over the screen boundary,
and choosing whether to wrap around or not;
1. Allowing the user to chose a display resolution, fullscreen, etc...

And some nice to have that I'd like to give a go at at some point:

1. Having a less boring render of the display (I just render white squares on a
black background) by adding some scanlines, distortion, and maybe glare to try
and emulate an old CRT;
1. Diving much deeper into clock speed and cycle synchronisation by actually
taking into account how long instructions are supposed to take and try to
replicate it faithfully.

## References

1. The most useful one, that describes all the CHIP-8 specs and instructions is
 [Cowgod's CHIP-8 Technical Reference](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM);
1. A great [test suite](https://github.com/Timendus/chip8-test-suite) for the
CHIP-8 in the repo, so you can check your emulator has a sane behaviour on
simple tests and narrow issues to some operations;
1. A [table of instruction's timings](https://jackson-s.me/2019/07/13/Chip-8-Instruction-Scheduling-and-Frequency.html), should you want to do more precise emulation.
