---
title: [Writeup] Nuit du Hack CTF Quals 2018 - AssemblyMe - Rev300
---

Hi,  
  
I didn't publish a CTF writeup since a while and now I'm back with a write-up which demonstrates how I solved the task in few minutes :p  
  
We were given a web page which asks for an authentification password.  
[http://assemblyme.challs.malice.fr](http://assemblyme.challs.malice.fr/)  
  
The goal is to reverse engineer the code and figure out the correct password.  
  
The steps I followed to solve the challenge were:  
  
[+] Read Webpage source code  
[+] Invoke the authentication function from Chrome console  
  
<!--more-->

```javascript
Module["asm"]["_checkAuth"]("maro")
returns 1761
Module["asm"]["_checkAuth"]
ƒ 35() { [native code] }

```

  
[+] Download the Wasm file [http://assemblyme.challs.malice.fr/index.wasm](http://assemblyme.challs.malice.fr/index.wasm)  
[+] Decompile the wasm file to a C file using **wasmdec**  [https://github.com/wwwg/wasmdec](https://github.com/wwwg/wasmdec)  
  
There are other alternatives but the wasmdec gave me the best result.  
  

```bash
$wasmdec -i index.wasm -o index.c -m

```

  
[+] Looking at memory dump file **index.c.mem.bin**, I saw "Obfuscation" and "Encryption" related strings... I thought it will be a pain in the ass but...I tried another way  
[+] Search for return value **1761** in **index.c** file only 2 occurences in **fn_25()** function  
[+] Seems that the function returns 1761 if the verification fails otherwise it returns 1690  
  

```javascript
...
if (    fn_47(local_0, 1616, 4);== 0) {
if (    fn_47(local_0 + 4, 1638, 4);== 0) {
if (    fn_47(local_0 + 8, 1610, 5);== 0) {
if (    fn_47(local_0 + 13, 1598, 4);== 0) {
if (    fn_47(local_0 + 17, 1681, 3);== 0) {
if (    fn_47(local_0 + 20, 1654, 9);== 0) {
global$1 = local_1;
return 1690;
} // <No else block>
} // <No else block>
} // <No else block>
} // <No else block>
} // <No else block>
} // <No else block>
...

```

  
**fn_47** seems to be a strncmp like function:  
[-] The first argument local_0 points to the input.  
[-] The second argument is the index in memory  
[-] The third atgument is number of bytes to compare.  
  
[+] I learned from index.js file that to access memory I can use **Module["buffer"]**  
[+] Get 4 bytes from memory at index 1616  
  

```javascript
new TextDecoder("utf-8").decode(Module["buffer"].slice(1616,1620))
"d51X"

```

  
[+] Dump the entire password  
  

```javascript
dec = new TextDecoder("utf-8")
dec.decode(Module["buffer"].slice(1616,1620)) + dec.decode(Module["buffer"].slice(1638,1638+4))+dec.decode(Module["buffer"].slice(1610,1610+5))+dec.decode(Module["buffer"].slice(1598,1598+4)) + dec.decode(Module["buffer"].slice(1681,1681+3))+ dec.decode(Module["buffer"].slice(1654,1654+9))
"d51XPox)1S0xk5S11W_eKXK,,,xie"

```

  
[+] Provide it to the web page gives  
[+] Authentication is successful. The flag is NDH{password}  
  
Maybe isn't the intended solution regarding the encryption and the obfuscation but anyway I managed to authenticate myself :p  
  
Flag: **NDH{d51XPox)1S0xk5S11W_eKXK,,,xie}**  
  
Cheers !
