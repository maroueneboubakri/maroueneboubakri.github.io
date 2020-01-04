---
title: DefCamp CTF Qualification 2017 - Chio - rev360 - Writeup
---

Hi,

The task worths 360 pts. I was the first solver to earn bonus points, but, I didn't like the fact of releasing hint few hours before the end of the CTF, since there were few teams already solved the challenge without hints and with a lot of guessing and this is totally unfair.

The hardest part of the challenge was to determine the target architecture of the provided binary file.

The given  [binary](https://dctf.def.camp/quals-2017-kalskflsafkl/public_crack.bin) is a Chip8 ROM file. This information was not mentioned in the task statement, however the task title gave me a hint about the target CPU.

Disassemble the binary with a Chip8 disassembler:

<!--more-->

```c
        start:
0x120a|         JP 522
0x6861|         LD V8, 97
0x7665|         ADD V6, 101
0x6675|         LD V6, 117
0x6e21|         LD V14, 33
        adr_522:
0x6002|         LD V0, 2
0x6102|         LD V1, 2
0xa04b|         LD I, 75
0xd015|         DRW V0, V1, 21
0xa00b|         LD I, 11
0x6103|         LD V1, 3
0xd011|         DRW V0, V1, 17
0x6102|         LD V1, 2
0x6007|         LD V0, 7
0xa032|         LD I, 50
0xd015|         DRW V0, V1, 21
0x600c|         LD V0, 12
0xa019|         LD I, 25
0xd015|         DRW V0, V1, 21
0x6011|         LD V0, 17
0xd015|         DRW V0, V1, 21
0x6016|         LD V0, 22
0xa023|         LD I, 35
0xd015|         DRW V0, V1, 21
0xa006|         LD I, 6
0x6105|         LD V1, 5
0xd011|         DRW V0, V1, 17
0x8120|         LD V2, V1
0x8230|         LD V3, V2
0x8340|         LD V4, V3
0x8450|         LD V5, V4
0x8730|         LD V3, V7
0x8990|         LD V9, V9
0x8561|         OR V5, V6
0x8320|         LD V2, V3
0x8440|         LD V4, V4
0x8652|         AND V6, V5
0x8322|         AND V3, V2
0x8563|         XOR V5, V6
0x8230|         LD V3, V2
0x8560|         LD V6, V5
0x8280|         LD V8, V2
0x8560|         LD V6, V5
0x8883|         XOR V8, V8
0x8450|         LD V5, V4
0x8540|         LD V4, V5
0x8320|         LD V2, V3
0x8650|         LD V5, V6
0x8441|         OR V4, V4
0x8433|         XOR V4, V3
0x8432|         AND V4, V3
0x8326|         SHR V3
0x8460|         LD V6, V4
0x8547|         SUBN V5, V4
0x8310|         LD V1, V3
0x8540|         LD V4, V5
0x8990|         LD V9, V9
0x8941|         OR V9, V4
0xf00a|         LD V0, K
0xf10a|         LD V1, K
0xf20a|         LD V2, K
0xf30a|         LD V3, K
0xf40a|         LD V4, K
0xf50a|         LD V5, K
0xf60a|         LD V6, K
0xf70a|         LD V7, K
0xf80a|         LD V8, K
0xf90a|         LD V9, K
0xfa0a|         LD V10, K
0xfb0a|         LD V11, K
0xfc0a|         LD V12, K
0xfd0a|         LD V13, K
0xfe0a|         LD V14, K
0xff0a|         LD V15, K
0x8014|         ADD V0, V1
0x8024|         ADD V0, V2
0x8034|         ADD V0, V3
0x8044|         ADD V0, V4
0x8054|         ADD V0, V5
0x8064|         ADD V0, V6
0x8074|         ADD V0, V7
0x8084|         ADD V0, V8
0x8094|         ADD V0, V9
0x80a4|         ADD V0, V10
0x80b4|         ADD V0, V11
0x80c4|         ADD V0, V12
0x80d4|         ADD V0, V13
0x80e4|         ADD V0, V14
0x3101|         SE V1, 1
0x1338|         JP 824
0x320d|         SE V2, 13
0x1338|         JP 824
0x330e|         SE V3, 14
0x1338|         JP 824
0x340c|         SE V4, 12
0x1338|         JP 824
0x3500|         SE V5, 0
0x1338|         JP 824
0x360d|         SE V6, 13
0x1338|         JP 824
0x370e|         SE V7, 14
0x1338|         JP 824
0x380d|         SE V8, 13
0x1338|         JP 824
0x390b|         SE V9, 11
0x1338|         JP 824
0x3a0e|         SE V10, 14
0x1338|         JP 824
0x3b0a|         SE V11, 10
0x1338|         JP 824
0x3c07|         SE V12, 7
0x1338|         JP 824
0x3d05|         SE V13, 5
0x1338|         JP 824
0x3e03|         SE V14, 3
0x1338|         JP 824
0xe0|           CLS
0x6002|         LD V0, 2
0x6102|         LD V1, 2
0xf229|         LD F, V2
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf329|         LD F, V3
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf429|         LD F, V4
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf529|         LD F, V5
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf629|         LD F, V6
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf729|         LD F, V7
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf829|         LD F, V8
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xf929|         LD F, V9
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xfa29|         LD F, V10
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xfb29|         LD F, V11
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xfc29|         LD F, V12
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xfd29|         LD F, V13
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0x0|            GOTO adr:0
        adr_824:
0xe0|           CLS
0x6002|         LD V0, 2
0x6102|         LD V1, 2
0x6f0b|         LD V15, 11
0xff29|         LD F, V15
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0x6f00|         LD V15, 0
0xff29|         LD F, V15
0xd015|         DRW V0, V1, 21
0x7005|         ADD V0, 5
0xd015|         DRW V0, V1, 21
0x0|            GOTO adr:0

```

  

Reffering to  [Chip8 Instruction set](http://devernay.free.fr/hacks/chip8/C8TECH10.HTM),

When the program starts, it draws PA557 on the screen, prompting to enter the password.

Using DRW instruction, it reads n bytes from memory, starting at the address stored in I. These bytes are then displayed as sprites on screen at coordinates (Vx, Vy). Sprites are XORed onto the existing screen.

Then using LD instruction, it waits for a key press, stores the value of the key in V1 to V14 registers.

After 14 key presses, the program compares each register V1 to V14 to an expected values, and if they are not equal, it jumps to label 824 where the program clears the display and draws a bad password message.

Hence, the first keypress stored at register V1 should be equal to 1, next keypress should be 13 and so on and so forth.

According to the documentation, computers which originally used the Chip8 language had a 16-key hexadecimal keypad. After several tries, emulating the ROM file, I built the keymap mapping each keycode to my keyboard.

```bash  
'1': 0,
'2': 1,
'3': 2,
'4': 3,
'q': 4,
'w': 5,
'e': 6,
'r': 7,
'a': 8,
's': 9,
'd': 10,
'f': 11,
'z': 12,
'x': 13,
'c': 14,
'v': 15,
```

Hence, the requested key presses are the following 2xcw1xcxfcdrz4f.

Entering this cheatcode should make the program jump to addr_522 label where the screen is cleared and a sprite series are loaded and printed to screen.

The string was  **DEC0DEDBEA75**.

Finally, as the task states, after find the flag, you should make it compatible yourself.

The flag is DCTF{SHA256("dec0dedbea75")} =  **DCTF{d3b9bd4dcdd5e697ee29c1dc8c9c4174558dd46dca22a7b7f3af5404c04164e2}**

That's all :)
