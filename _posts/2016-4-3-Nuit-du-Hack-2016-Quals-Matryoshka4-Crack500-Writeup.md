  
Hi, I'm member of SpectriX tunisian CTF team. We were ranked #31 in this CTF because we played only 2 guys. I hope we will do better next CTF ;)

This task was the 4th level of the Crack series tasks of NDH. The previous levels were quiet easy except the last level which was tricky(as they said).

```bash
file stage4.bin
stage4.bin: DOS/MBR boot sector
```


The given binary is a bootloader. To run it I used qemu.

```bash
qemu-system-x86_64 -fda stage4.bin
```

After booting the program asks for a magic word. Let's go further with debugging.

 
The -s option from Qemu is an alias for -gdb tcp::1234. This will setup Qemu to listen on port 1234 and wait for a gdb connection to it. The -S option makes Qemu stop execution until we provide the continue command c. Let's execute again the bootloader and attach GDB to it

```bash
qemu-system-x86_64 -fda stage4.bin -s -S
gdb
(gdb)target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
0x0000fff0 in ?? ()
c
```

I tried to print each executed instruction and its dissassemblyin order to have a global idea about the execution trace

```bash
(gdb)set logging on
(gdb)set height 0
(gdb)while 1
> x/i $pc
> stepi 
> end
```

That was not so helpful so I decided to proceed with static analysis. The bootloader is a RAW binary and to disassemble it I used ndisasm

```bash
ndisasm -b 16 stage4.bin
```
 
The bootloader uses BIOS interrupts and that's why I set the processor mode to 16 bits.

After getting the disassembly file I tried to find CMP instructions inside it.


Instructions starting from the 0x6B3 offset attempt to compare the content of the CL register with printable ASCII chars which form the Good_Game_! string. I thought that I found the flag and that this task does not diserve 500 pts at all :D. I restarted the bootloader and entred the flag. I got "OK...You win this time..." I entred the flag in the platform but it has been rejected.

Maybe it is a side effect...Maybe it is part of the challenge...Obfuscation...But in all cases a program must accept only one string in input which is the flag.

Forget about that. Let's read again the disassembly, try to identify which parts of the bootloader is the validation routine. The idea which came across my mind is to search for instructions blocks that try to load bytes from memory and do baisc operations on it.

 
I found the following block:

```assembly
0x69b:   57                     push edi
0x69c:   50                     push eax
0x69d:   56                     push esi
0x69e:   53                     push ebx
0x69f:   83 f8 00             cmp eax, 0x0
0x6a2:   74 19                  jz  0x6bd 
0x6a4:   8a 0f                  mov cl, byte [ edi ]
0x6a6:   8a 16                  mov dl, byte [ esi ]
0x6a8:   30 d1                  xor cl, dl
0x6aa:   88 0f                  mov byte [ edi ], cl
0x6ac:   48                     dec eax
0x6ad:   4b                     dec ebx
0x6ae:   47                     inc edi
0x6af:   46                     inc esi
0x6b0:   83 fb 00             cmp ebx, 0x0
0x6b3:   7f ea                  jg  0x69f

```

This block is used to decrypt the output messages

and it is also used to encrypt the input flag.

The encryption/decryption key is :

```
6C 53 05 6A 5C FC FB 0E AD 4A B9 93 AD 3D 16 33 14 DE 45 4A 12 78 E8 A0 FF
```
  
To find out how the flag is XORed I relied on Hardware breakpoint on read.

```bash
(gdb) target remote 127.0.0.1:1234
Remote debugging using 127.0.0.1:1234
0x0000fff0 in ?? ()
(gdb) rwatch *0x1000
Hardware read watchpoint 1: *0x1000
(gdb) c
Continuing.
Hardware read watchpoint 1: *0x1000

Value = 1895938025
0x000014ab in ?? ()
(gdb) i r edi esi
edi   0x19fc 6652
esi   0x19cb 6603

(gdb) x/26x $esi
0x19cb: 0x6c 0x53 0x05 0x6a 0x5c 0xfc 0xfb 0x0e
0x19d3: 0xad 0x4a 0xb9 0x93 0xad 0x3d 0x16 0x33
0x19db: 0x14 0xde 0x45 0x4a 0x12 0x78 0xe8 0xa0
0x19e3: 0xff 0x00

(gdb) c
Continuing.
Hardware read watchpoint 4: *0x1000

Value = 1090631657
0x00001472 in ?? ()
(gdb) x/i $pc
=> 0x1472: mov    (%esi),%dl
(gdb) set disassembly-flavor intel
(gdb) x/i $pc
=> 0x1472: mov    dl,BYTE PTR [esi]
(gdb) stepi
0x00001474 in ?? ()
(gdb) x/i $pc
=> 0x1474: xor    cl,dl
(gdb) i r cl dl
cl    0x41 65
dl    0x6c 108
(gdb) stepi
0x00001476 in ?? ()
(gdb) x/i $pc
=> 0x1476: mov    BYTE PTR [edi],cl
(gdb) 

(gdb) i r edi
edi  0x1003 4099
(gdb) 

```

  

and so on ...
  
    flag[0x1003] ^ 0x28 = 0x6c => 0x44
    flag[0x1004] ^ 0x37 = 0x53 => 0x64
    flag[0x1005] ^ 0x77 = 0x05 => 0x72
    flag[0x1006] ^ 0x5b = 0x6a => 0x31
    flag[0x1007] ^ 0x31 = 0x5c => 0x6d
    flag[0x1008] ^ 0x90 = 0xfc => 0x6c
    flag[0x1009] ^ 0xd4 = 0xfb => 0x2f
    flag[0x100a] ^ 0x68 = 0x0e => 0x66
    flag[0x100b] ^ 0xdf = 0xad => 0x72
    flag[0x100c] ^ 0x2c = 0x4a => 0x66
    flag[0x100d] ^ 0xb9 = 0xb9 => 0x00

Flag:  **Ddr1ml/frf**
