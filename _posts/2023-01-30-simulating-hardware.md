---
title: Simulating Hardware
author: DidierMalenfant
redirect_from: /blog/simulating-hardware
---
I'm slightly behind on blogging about the current progress. I'm not working on this full time but progress is already going faster than I expected. Let's catch up...

When I left it last week I had a blank window opened in the simulator. Not much and yet a very important milestone because everything else can build on this while keeping the project in a run-able state at all times.

The purpose of the simulator is two-fold. One it's to help rapid-prototype the **SDK** and the platform using more powerful tools than what the **FPGA/Pocket** combination will provide. Running under **XCode**, I get a professional grade **IDE** complete with easy debugging. Second, it will probably turn into a critical part of the **SDK** allowing developers to test their code without the actual hardware. There's nothing that stops me down the road from turning it into a generic player for the games. I could even make builds of that on **iPhone**, **iPad**, **Apple TV**, etc...

Main goal though is to have it replicate the actual console as close as possible from the real thing and that mean not really developing it like a real software application. Instead, I'm going to code each subsystem separately and connect them inside the app using just memory accesses, like the real hardware would, instead of calling methods. I'll have the **CPU** running game code, the Memory Management Unit (**MMU**) directing memory read/writes, the Graphics Processing Unit (**GPU**) drawing things to **VRAM** and finally the screen showing it to you.

So let's start with the **CPU**. I don't know much about **FPGA** yet, although I've started [reading up](https://books.apple.com/us/book/fpga-programming-for-beginners/id1546170695) on it, but it seems to me like the kind of **CPU**s achievable on a **Cyclone V** is going to be limited by the number of logic elements available. It's 49K on the Pocket. I'm not sure what that translates to in terms of **CPU** complexity but I'm pretty sure I'm not going to put an **M2** chip on that. **ARM** does provide [FPGA cores](https://www.arm.com/resources/free-arm-cortex-m-on-fpga) for some of their **CPU**s but I want to keep everything open source and I think this level of complexity may be a bit ambitious to begin with. I can always do a **V2** console down the road with a better **CPU**.

What should I pick? What about a good old **MC68000**? It's powering a lot of the retro cores so I know it will run pretty well. I know this little gem by heart because I learned to program on it back in the 80s on my **Amiga 1000**. And best of all, there is an [open-source core](https://github.com/ijor/fx68k) for it and an [open-source emulator](https://github.com/kstenerud/Musashi) that I can use in **pfSimulator**. I hope this can run fast enough because if it does, it means I can use the exact same binaries on the console and in the simulator which is nice. I've never interfaced with or bootstrapped an **MC68000** in hardware but that seems way more approachable than an **ARM-Cortex** and there is some [good documentation](https://www.amazon.com/Microprocessor-Systems-Design-Hardware-Interfacing/dp/0534948227/ref=sr_1_1?crid=ASQZCJB9QDY9&dchild=1&keywords=alan+clements&qid=1594379040&sprefix=alan+clem%2Caps%2C165&sr=8-1) on it. So **MC68000** it is then.

If I want to plug the emulator into **pfSimulator**, I'll need a binary file to run and I don't have a toolchain yet. That's on deck for this week though but for the time being, I will emulate the...err...emulator by writing some code poking at the memory via the **MMU**. What will the code do and more importantly what will our **GPU** do for now? Let's keep it simple and let the **GPU** change the screen's background color. For that I need to be able to set a color and tell it to clear the screen. All this will be done via the **GPU**'s registers, which I named **PFX-1**, and that I arbitrarily placed in my memory map:
```
// -- Register Address Map
#define PF_PFX1_BASE                       0x00D00000
#define PF_PFX1_VSYNC_COUNT                0x00000000
#define PF_PFX1_CONTROL                    0x00000002
#define PF_PFX1_COLOR_RG                   0x00000004
#define PF_PFX1_COLOR_BA                   0x00000006

// -- Register Content
#define PF_PFX1_CONTROL_CLEAR_SCREEN       0x0001
#define PF_PFX1_CONTROL_SWAP_BUFFER        0x0002
```
The **MC68000** has a 16-bit bus so I will make our registers 16-bit also. I have a control register where I can write commands, two registers for setting the color (Red, Green then Blue and Alpha respectively), and also some housekeeping things like a **VSync** counter so I can wait for the next **VSync**. Commands are simple, clear the screen or swap the render and display buffers (our **VRAM** is double buffered rigth now).

The goal is to have this **PFX-1** 'chip' work in a separate thread and only communicate via those registers, like real hardware would. Our **CPU** won't even directly poke in those registers because I'm going to insert an **MMU** in between those two to arbitrate our virtual data bus. What this means is that if the **CPU** wants to read/write to the **PFX-1** chip it, it talks to the **MMU** providing the register's address, `0x00D00002` for example. The **MMU** knows that things starting at `0x00D00000` are **PFX-1** things and so it directs the read/write to that.

This is of course **NOT** the most efficient way of doing this just to clear the screen a certain color, but what I want here is to replicate the hardware so that I can build the firmware and the **SDK** around this.

Once I have all this setup, I can write a simple little code loop, even without an emulated **CPU** yet, to test out the system:
```
PFCpu* this = (PFCpu*)data;

byte r = 0;
byte g = 0;
byte b = 0;

while (!this->thread_should_exit) {
    pfMmuWrite(this->mmu, PF_PFX1_BASE + PF_PFX1_COLOR_RG, (r << 8) | g);
    pfMmuWrite(this->mmu, PF_PFX1_BASE + PF_PFX1_COLOR_BA, (b << 8) | 255);
    pfMmuWrite(this->mmu, PF_PFX1_BASE + PF_PFX1_CONTROL, PF_PFX1_CONTROL_CLEAR_SCREEN);

    r += 12;
    g += 4;
    b += 34;

    uint16 previous_vsync = pfMmuRead(this->mmu, PF_PFX1_BASE + PF_PFX1_VSYNC_COUNT);
    while (pfMmuRead(this->mmu, PF_PFX1_BASE + PF_PFX1_VSYNC_COUNT) == previous_vsync) {
        if (this->thread_should_exit) {
            return 0;
        }

        SDL_Delay(1);
    }

    pfMmuWrite(this->mmu, PF_PFX1_BASE + PF_PFX1_CONTROL, PF_PFX1_CONTROL_SWAP_BUFFER);
}
```
This simply sets a color, clears the screen, waits for the next **Vsync** and flips the display buffer. Remember this runs on the host (my **MacBook** in this case) and not on a simulated **CPU** yet. The results are lovely (I slowed down the **VSync** on purpose here in order to avoid too many flashing colors on the web site):
<div style="text-align: center;">
    <img src="/assets/blog/2023-01-30/Simulator-clear-screen.gif" alt="PfSImulator window with the screen flashing colors" width="60%">
</div>
<br>

If you want to follow along, the repo for **pfSimulator** can be found [here](https://github.com/ProjectFreedomGaming/pfSimulator). This week's progress is at commit [744f636e](https://github.com/ProjectFreedomGaming/pfSimulator/commit/744f636ea34f514d207240e4d5f0e728cd9b48bc).

With ❤️ from Paris, France.
