---
author: Pratyush Nair
title: Writing a Chip8 VM
date: 2023-07-24
description: Writing a Chip8 VM
---

Chip8 is an interpreted programming language originally made to run on COSMAC VIP and Telmac 1800 machines. Chip8 programs run on the Chip8 virtual machine. It was designed to run video games and it's simple design makes it easy target for emulation, which is exactly why I started with this! Unlike other popular choices like NES and GameBoy, emulating Chip8 does not involve emulating actual hardware but a virtual machine, which is much easier for a beginner. 

Chip8 was later expanded to allow execution of more and more advanced programs. Some of these advancements are *Chip48* and *SCHIP*. However, I'm focusing solely on the original Chip8 specs.


> Disclaimer: The entire VM can be implemented even by using Wikipedia as the only reference (except maybe the drawing algorithm). That should give you an idea on how well documented this architecture is, so I will not bother listing the instructions or explaining them. This is solely a devlog of the mistakes I made while implementing this and scope of further development.


Chip8, in brief:
* Whopping 4K of RAM
* 16 8-bit registers
* Stack, purely for function calls
* Sound and delay timer
* Input which recognized 16 keys from 0 to F
* 64\*32 pixel monochrome display buffer
* 35 opcodes each 2 byte long

I chose to write the VM in C++, using SDL as the media layer. Since the display is monochrome, rendering it as easy as copying the raw pixel data in the buffer to the render texture. Decoding the instruction is straightforward. Without exception, all the instructions are 2 bytes long. Instead of using function pointers, I went with a simple `switch` for decoding the instruction. 

Early in the process, I implemented logging to track the execution of each instruction. This saved me a lot of pain and suffering later on. One such instance was when I got the draw instruction wrong. Here is how the draw instruction (DXYN) works:

* Sprite data is fetched from the memory from the location stored in index register
* X and Y coordinates are encoded in the instruction itself
* Sprites are XORed onto the existing screen

Remember how I talked about rendering raw pixel data? That works because the display is monochrome and each pixel is either 0xFFFFFFFF or 0 which corresponds to black or white respectively. The bug caused visual anomalies while rendering, and this was because I was comparing sprite data(0s and 1s) to the video buffer(0s and 0xFFFFFFFFs)! 

Another interesting issue was with painfully slow framerate. Initially, the main loop executed without any delays. 

```cpp
while(!kill) {
  kill = platform.read_input(keypad);
  vm.execute_cycle(keypad);
  platform.render(vm.get_video_memory());
}
```

While this worked for the test ROMs, it did not for actual games. Minor issue, I thought and I capped the FPS at 60. However, this was still slow! After a bit more digging, it seems that I found out what I was missing: Chip8 executes best at 500-700Hz. Different Chip8 games had different frequencies, so this had to be given as the input along with the ROM file name.

```cpp
while (!kill) {
  kill = platform.read_input(keypad);

  Uint64 start = SDL_GetPerformanceCounter();
  for (uint16_t i = 0; i < frequency/FPS; i++)
    vm.execute_cycle(keypad);
  Uint64 end = SDL_GetPerformanceCounter();

  auto elapsedMS = ((end - start) * 1000) / SDL_GetPerformanceFrequency();

  SDL_Delay((1000.0/FPS) > elapsedMS ? ((1000.0/FPS) - elapsedMS) : 0);
  platform.render(vm.get_video_memory());
  vm.update_timers();
}
```

Note, delay and sound timers executes at 60Hz, so they had to be update separately. 

Test ROMs by [Timendus](https://github.com/Timendus) also helped me track down a few other minor issues, also related to display. As of now, most "old" ROMs work. Next time I write an emulator, I plan on implementing a debugger UI first. This seems much saner than reading through logs or stepping through code in an external debugger. Being able to step through the actual ROM binary would be a huge blessing, especially for more complicated systems.

Even though I planned to complete this within a single day, it took me two days to iron out all the bugs. Despite this, I plan on writing another emulator soon enough. Maybe 6502? Or maybe take it a step further and write a NES emulator? Who knows.

