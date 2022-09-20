+++
title = "Rust & Wasm CHIP-8 Emulator"

[taxonomies]
tags = ["rustlang", "emudev"]

[extra]
location = "San Francisco/Lisbon"
+++

What started as an attempt to demonstrate how interrupts and grayscale
rendering works on the TI series graphing calculators turned into a
full-blown attempt at writing an emulator that would be runnable in a
modern browser using a combination of [Rust][rust-lang] 🦀 and
[WebAssembly][wasm] 🕸. The idea came to me while walking through the
[Rust + WebAssembly tutorial][rust-wasm], where I realized that many of the
same abstractions could apply to an emulated system.

<!-- more -->

[rust-lang]: https://doc.rust-lang.org/book/second-edition
[rust-wasm]: https://rustwasm.github.io/book/game-of-life/introduction.html
[wasm]: https://developer.mozilla.org/en-US/docs/WebAssembly

I should note before you get any further that this is not a full guide to
writing an emulator in Rust. I just wanted to document some interesting
implementation details during my implementation, but this should (hopefully) act
as a short “Getting Started” guide to point you in the right direction of your
own emulator or if you just wanted to see what a Rust + WebAssembly project
looks like.

## Let's Play

I've embedded the emulator below so that you can try it out before reading
any further. The keyboard for the CHIP-8 is represented by the 4x4 grid
below.

```
Normal Keyboard      CHIP8 Keyboard
| 1 | 2 | 3 | 4 | -> | 1 | 2 | 3 | C |
| Q | W | E | R | -> | 4 | 5 | 6 | D |
| A | S | D | F | -> | 7 | 8 | 9 | E |
| Z | X | C | V | -> | A | 0 | B | F |
```

Just select one of the ROMs from the dropdown menu, hit the play button
and it'll start the emulator.

<iframe scrolling="no"
        style="border: 1px solid #AAA; height: 256px; border-radius: 8px;"
        src="https://a5huynh.github.io/rusty-emulator/chip8/">
</iframe>


## CHIP-8: Hello Emulator

To dip my toes into emulator development, I decided to start small.
[CHIP-8][chip8] is often considered the "Hello World" equivalent of
emulator projects due to its simplicity, with only 35 instructions, simple
keyboard input, and simple sound management.

The CHIP-8 was never actually a real chip/system and was developed to be more
of a virtual machine that could be run on different microcomputers at the
time. It nevertheless captures all the abstractions required for other
emulator projects and offers a friendly dip into the emulator development
world. Having existing emulators to compare with was also tremendously
helpful to understand certain subtleties of the implementation.

[chip8]: https://en.wikipedia.org/wiki/CHIP-8

Additionally, there are also tons of great resources that fully describe
the instruction set and even walk you through an implementation:

- [/r/EmuDev Subreddit][emudev]
    - Tremendously helpful just looking at old posts and seeing what others have
      done.
- [Cowgod's Chip-8 Technical Reference][cowgod]
    - Pretty much my go-to reference for each opcode that is implemented in the
      project.
- [How to write an emulator (CHIP-8 Interpreter)][howto]
    - I didn't delve into this blog post too much, but I did use it to compare
      certain implementation details that were giving me trouble.

[emudev]: https://www.reddit.com/r/EmuDev/
[cowgod]: http://devernay.free.fr/hacks/chip8/C8TECH10.HTM
[howto]: http://www.multigesture.net/articles/how-to-write-an-emulator-chip-8-interpreter/


### Setup

If you've followed the Rust-Wasm Book tutorial already, the following
commands will look familiar. I started the project by generating the
project folder with everything necessary to get started:

``` bash
# Generates the rust project folder
> cargo generate --git https://github.com/rustwasm/wasm-pack-template emu-project
# Go into the project directory to initialize the javascript project
> cd emu-project
> npm init wasm-app www
```

Keeping with the statically typed language theme, I wanted to use
[TypeScript][typescript-lang] for as much of the browser code as possible,
which required a couple tweaks to the generated project:

* Added a `tsconfig.json` file to configure the TypeScript compiler.
* Updated the `package.json` file to include TypeScript dev dependencies.
* Updated the `webpack.config.js` file to compile TypeScript and hot module
  reload when it detects any changes.

As for the generated javascript files output by `wasm-pack`, I couldn't
quite figure out how to also convert `bootstrap.js` and `index.js` files to
TypeScript so these are kept the same and will import the rest of the
application.

[typescript-lang]: https://typescriptlang.org


### What do we need to emulate?

Outside of the instructions, we'll also need to emulate essential parts of
the CHIP-8 system that are required to run a program.

* Various [registers](https://en.wikipedia.org/wiki/Processor_register)
    * 15 8-bit general purpose registers.
    * 1 8-bit carry register used as a “carry” flag during arithmetic.
    * 1 8-bit stack pointer (often referred to as the `SP` register).
    * 1 16-bit program counter (often referred to as the `PC` register).
    * 1 16-bit index register.
* 4096 bytes of memory.
* 2048 bytes to represent the display, creating a 64 x 32 pixel display.
    * This actually only needs to be represented as a 0 or 1, so an array
      of booleans would work as well.
* A sound & delay timer.

Most if not all of these can easily be represented by arrays of the the
aforementioned types, all of which can be unsigned integers.


### Execution Loop

At a high level, for successful emulation of the CHIP-8 system we want to
correctly imitate every cycle of the CPU, keep track of any changes to the
display/memory/registers, and finally handle any keyboard input.

If we had a function that would handle each `tick` of the CPU, it would
look something very similar to the following:

``` rust,linenos
fn tick(&mut self) {
    // 1. Find the opcode pointed to by the `PC` register.
    let opcode = self.fetch()
    // 2. Fetch and decode the opcode.
    // 3. Execute the opcode & update the `PC`.
    self.execute(opcode);
}
```

This simple execution loop will form the basis for the rest of emulation
code. Most, if not all, of our logic will happen inside the `execute`
function, where each opcode will be decoded and applied to the
registers/memory/display.


#### What happens during `fetch()`?

At a very high level, fetch returns a 16-bit instruction pointed to by the
`PC` register and increments the `PC`. The only “gotcha” here would how
instructions are stored in memory, [most-significant byte][msb]
first. Taking this into account, the code to grab the opcode in the correct
byte order would like the following:

[msb]: https://en.wikipedia.org/wiki/Bit_numbering#Most_significant_byte

``` rust,linenos
let opcode = u16::from(mem[pc]) << 8 | u16::from(mem[pc + 1]);
```


#### What happens during `execute()`?

Like mentioned before, `execute()` is where the majority of the logic for
the emulator will reside. In the case of my implementation, I break the
opcodes into different prefixes (`0x1000`, `0x2000`, etc) and implemented the
logic for each corresponding prefix. For example, lets take the simple `0x00E0`
opcode that is used to completely clear the screen:

``` rust,linenos
// First, we grab the first bytes of the opcode as a prefix
let prefix = opcode & 0xF000;
// Grab the lower two bytes.
let lower = (opcode & 0x00FF) as u8;
match prefix {
    // Match and handle 0x0xxx opcodes
    0x0000 => {
        match lower {
            // Clear display.
            0xE0 => {
                for idx in 0..DISPLAY_SIZE {
                    self.display[idx] = 0;
                }
            }
        }
    },
    // handle 0x1xxx opcodes
    0x1000 => {},
    // etc.
    ...,
    _ => log!("Unknown opcode {:#X}", opcode)
}
```

The opcode prefixes can be grabbed using a little bit-slinging magic that
you see at the top of the code example: `opcode & 0xF000`. This bitwise `AND`
operation will only return the very first byte of the opcode. This made it
easy to write tests for each opcode, since instructions in the same prefix
tend to be related to each other.


## Rendering the Display

While most of the opcodes implemented in `execute()` tend to be only a
handful of lines, the most complex of the opcodes is used to draw sprites
to the screen. This single instruction handles reading a sprite stored in
memory, xor-ing it to display memory. Additionally, a flag is also set if
any pixels are _erased_, a feature is that is often used in programs for
collision detection.

The 64x32 pixel display has a coordinate system that starts at the top left
and extends downwards:

```

┌─────────────────┐
│(0,0)         (63,0)│
│                    │
│(0,3)        (63,31)│
└─────────────────┘

```

Sprites that cross horizontal/vertical boundaries are wrapped to the other
side. For example, if we have a sprite that starts at `(63,0)`, the pixel
at `(63,0)` will be set on and then wrapped around to `(0,0)`, `(1,0)`,
etc.


## Handling Input

The CHIP-8 system uses a hex keypad, which is a little odd but can be
mapped easily to any 4x4 set of keys on modern day keyboards. All that
needs to be tracked is whether key is pressed and whether the key was the
last pressed key. This is accomplished by adding an event listener for the
`keyup` and `keydown` events, where the key press is sent to the emulator
for handling.

Here is an example of the `handleKeyPress` code for both the typescript
and rust side. The typescript side needs a key map thats not shown
to map the pressed key code to the corresponding emulated key, discarding
all other key presses.

``` typescript
public handleKeyPress(ev: KeyboardEvent) {
    if (ev.keyCode in KEY_MAP) {
        this.engine.key_press(KEY_MAP[ev.keyCode]);
    }
}
```

And for the rust side, we keep track of the currently pressed key as well
as setting the key to pressed until we receive a release event for that key.

``` rust,linenos
pub fn key_press(&mut self, key: Key) {
    self.current_key = Some(key);
    self.keys[key as usize] = true;
}
```


## Up Next

And that's pretty much it! I could've gone into far much depth into things
like the opcodes or perhaps the display rendering, but I leave that up as
an exercise for the reader. All the code for this emulator will be publicly
available along with the research and ROMs used during its development.

Note that at the time of this writing there are still currently some bugs
with the more complex ROMs and potentially some speed improvements that could
be done. I plan on fixing up the last remaining bugs w/ the emulator and
implementing the basic sound handler.

The next emulator added to the mix will probably be a Z80 or NES emulator,
a much larger emulation implementation but should be able to reuse many of
the components created for this project.

> You can follow progress on my emulator(s) development [here][rust-emu].

[rust-emu]: https://github.com/a5huynh/rusty-emulator
