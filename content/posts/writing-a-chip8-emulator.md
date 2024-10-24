+++
title = 'Writing a CHIP-8 Emulator'
date = '2024-10-24T15:34:57+02:00'
draft = true
toc = true
tocBorder = true
[params]
  author = "Jordan Emme"
+++

## CHIP-8

### What Is It?

The CHIP-8 is an interpreted language created by Joseph Weisbecker, to run on a
1802 microprocessor. It provides a simple and lightweight language to code with
on these older chips. Whilst not being an actual, physical machine, writing an
interpreter for the CHIP-8 is seen as the "Hello World!" of emulation, as it is
designed as a virtual machine, which comes with its own display, memory layout,
CPU architecture and instructions. As such, let us not be pedantic and delve
into how to write a CHIP-8 emulator on a modern machine.

### Specifications

#### "Physical" aspect

The CHIP-8 consists of:
* A CPU that typically runs at 700Hz (although implementations may vary);
* 16 registers (15 general purpose variables + 1 flag);
* A program counter;
* An index register to point at locations in memory;
* A stack (usually 32 bytes but may vary);
* A whopping 4096 bytes of memory;
* An 8-bits delay timer;
* An 8-bits sound timer;
* A 60Hz, 64x32, monochrome display.

That is all the "hardware" that we have to emulate. This is all quite
straightforward as we will see in the next sections.

#### Other Prerequisite

On top of that hardware, the CHIP-8


## How to Emulate

### Data Needed

### Fetch-Decode-Execute Cycle

### The Display

### Clock Speed

## Testing

## References
