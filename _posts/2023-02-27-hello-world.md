---
title: Hello, World!
author: DidierMalenfant
redirect_from: /blog/hello-world
---

I'm long overdue for an update and as you can imagine, a lot has happened since the last time we talked.

I left the project last time with a working **simulator** app, running on **macOS** and simulating the **CPU**, the graphics chip, its registers and the screen itself. It wasn't doing much but it was changing the screen's background color every frame.

Well, the current state is...just about the same.

Don't be disappointed just yet. It looks the same in terms of what the **simulator** shows on screen but under the scenes it's now almost completely different. I now have an **SDK**, a **firmware** and an emulated **68000** CPU!

### The toolchain

The first part of the **SDK** was getting a custom build **GCC** toolchain to compile **c** code into **68000** assembly. **GCC** already supports **68000** as a target, so all I needed was to get it building from source and include the result in the **SDK**. But since I wanted to be able to compile regular c code, I also needed my own version of **libc** (which, for the purpose of this conversation include **libgcc** and **stdlib**) in order to have access to things like `printf()`, `strcpy()` or `malloc()`.

Luckily, I stumbled across this great [video](https://www.youtube.com/watch?v=E-TZBDJVggA) from **Tom Storey** discussing how he put together a bare bones **68000** toolchain to compile **C** for his dev board. The result is [m68k_bare_metal](https://github.com/tomstorey/m68k_bare_metal) and that became a great starting point for my own toolchain.

I want as much of the project as possible to be available as source code and be easy to rebuild from source so I forked both **GCC** and **binutils**'s codebases and added them to repos under the [Project Freedom](https://github.com/ProjectFreedomGaming)'s organization on Github. This helps make sure that I will keep around all the code that was used to build any dependencies for the project without having to hunt it all over the internet. I've also started creating build scripts to easily rebuild all those from scratch, in order to keep a memory of how things work and save some time should someone want to it themselves in the future. While I was at it, I did the same for the **SDL** library which the simulator uses, and will continue to do so for any other future dependency.

The master script to build the toolchain is written in **Python** and it checks that you have all the right tools and libs available (via homebrew), then clones the repos for **binutils** and **GCC** and builds them. There is even an option to install the result automatically into the **SDK** folder.

Unfortunately for now it seems like **GCC** can't be built as a universal binary macOS because of some its dependencies. I'll need to compile **mpfr**, **libmpc** and **gmp** myself as universal binary for this. This sounds like fun but after dabbling into it for a bit, I decided this was better left for later.

Most of **libc** was taken from **m68k_bare_metal**, apart from some Apple-contributed code that wasn't compatible with **GPL-v3**. For those I hunted down replacements and made a note to test them down the road to make sure they work correctly.

The **makefile** system and **linker** scripts are also derived from **m68k_bare_metal**, although I tweaked the **makefile** quite a lot to make it easier to use for end-user projects. A **makefile** can now be as simple as:
```
# -- Name of the rom we are building.
ROM_TARGET = helloworld.pfxrom

# -- Finally the common makefile should be included last.
include $(PF_SDK_ROOT)/src/common.mk
```

With that I was able to compile a rom file, complete with real **68000** assembly, to perform a very simple and traditional task:
```
#include <stdio.h>

int main(void)
{
    printf("Hello, World!\n");

    // -- main() cannot return right now.
    while(1) {
    
    }

    return 0;
}
```

### Emulating the 68000

All that was left now was to get the **simulator** to read and run that file. Easy enough, right?

Once again some existing projects helped a lot here. I was able to plug in [Karl Stenerud](https://github.com/kstenerud)'s [MUSASHI](https://github.com/kstenerud/Musashi) emulator and the simply connected it to my simulated memory where the rom file had been loaded. All you have to so is server the read and write memory accesses that the **68000** requests and the emulator take cares of the rest.

### Connecting the dots on the simulator

I made the **SDK**'s **libc**'s `printf` write its output to a register I created in the memory map just for that:
```
// internal putchar_ wrapper
static inline void out_putchar(char character, void* buffer, size_t idx, size_t maxlen)
{
  (void)buffer; (void)idx; (void)maxlen;
  if (character) {
    pf->ioPutChar(character);
  }
}
```

Now this here is already interesting because **libc** is using the first few functions of what will become the **pfx**'s firmware. It calls `pf->ioPutChar()`, which in returns pokes into my **IO** register on the hardware:
```
void ioPutChar(char c)
{
    PF_WRITE_WORD(PF_IO_SYM_LOG_CHAR, c);
}
```

Once the code is emulated on the **simulator** via **MUSASHI**, writing the word inside the register just ends up calling this method:
```
void pfIOWriteWord(pointer address, word value)
{
    switch (address - PF_CUSTOM_CHIPS_BASE) {
        case PF_IO_SYM_LOG_CHAR: {
            printf("%c", value & 0xff);
            break;
        }
        default:
            // -- Illegal read address
            PF_ASSERT(false);
    }
}
```

which prints the character to **stdout**.

Apart from being a tradition when writing first programs, this will be very useful for debugging **68000** code un the simulator until I can plug in a real debugger at some point.

### Bootstrapping

Bootstrap code is linked in the **rom**. It provides the two pointers needed to initialize the **68000** on launch. It's then directed to the code itself which setups up and clear the **BSS** section as well as initializes any global variables and then jumps into `main()`. For the time being I'm literally jumping into `main()` instead of branching so this can never return. I will revisit later on.

### Giving it a spin

It was time to get all this stuff rolling and see what happens. Turns out there weren't many issues apart from two boneheaded errors on my part:

1. The **68000** is big-endian. **Apple Silicon** chips aren't. When accessing **word** or **long word** in memory from the simulator, this matters... 🤣
2. Accesses to memory addresses that are, in fact, registers can be optimized away by the compiler unless you use the `volatile` keyword for the address. Say you write or read to/from the same address five times, the compiler will very often only keep the last access and completely remove the first 4. Now that is fine if you're accessing a normal address which is not supposed to change unless you change it. But if that address is actually part of another chip, it could change and its value can't be take for granted. Adding the `volatile` to my register access macros fixed that:

```
#define PF_READ_WORD(offset)            *(volatile word*)(PF_CUSTOM_CHIPS_BASE + (pointer)offset)
#define PF_WRITE_WORD(offset, value)    *(volatile word*)(PF_CUSTOM_CHIPS_BASE + (pointer)offset) = value
```

The joy of seeing characters printed in the console! Another very important tracing bullet was done. I now have a tool chain which can build code into a rom, using a standard **c** library and an **SDK** if necessary. That **rom** can then be executed on the simulator where the code is bootstrapped and then emulated in real time. That's a lot of stuff happening. The only thing missing here is a real emulation of the graphic chip's hardware but everything else is pretty damn close to how the final thing will work.

### Meanwhile, back with the screen color sample code

I was finally ready to add some functions to the firmware for the graphics chip and rewrite the demo that changes the screen color in order to use these new methods. It now looks like this:
```
#include <pfSDK/Api.h>
#include <pfSDK/Types.h>

int main(void)
{
    byte r = 0;
    byte g = 0;
    byte b = 0;
    
    // -- main() cannot return right now.
    while(1) {
        pf->pfxWaitVSync();

        pf->pfxSetClearColor(r, g, b);
        pf->pfxClearScreen();
        
        r += 12;
        g += 4;
        b += 34;

        pf->pfxSwapBuffers();
    }

    return 0;
}
```

And guess what?? it **works**! Pretty much does the same thing as last time, changes the screen color, but it does a lot more under the hood to replicate the final hardware.

If you want to follow along, here are the repos for [pfSimulator](https://github.com/ProjectFreedomGaming/pfSimulator) and the [pfSDK](https://github.com/ProjectFreedomGaming/pfSDK). This week's progress is at commits [662605f](https://github.com/ProjectFreedomGaming/pfSimulator/commit/662605fac60aaa3e79d2a70d461a18eab8977a73) and [8056cca](https://github.com/ProjectFreedomGaming/pfSDK/commit/8056ccad012e8d8625ce6fb31c93faed66556c26). You can also follow my current [tasklist](https://trello.com/b/ihs3W4ux/%F0%9F%91%BE-project-freedom) on Trello.

With ❤️ from Paris, France.
