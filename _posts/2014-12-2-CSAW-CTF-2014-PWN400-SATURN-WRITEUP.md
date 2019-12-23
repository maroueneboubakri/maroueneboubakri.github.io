Hello, 

The task was about bypassing a challenge response authentication system, without having access to the library that check the submittet response. The first run of the binary gives a message telling that the libchallengeresponse.so is missing.

<!--more-->

```bash
./saturn: error while loading shared libraries: libchallengeresponse.so: cannot open shared object file: No such file or directory
```

So, I managed to create the missed library and functions. I imported the binary into IDA and i got a loook at the Imports list. fillChallengeResponse() is an extern function loaded from the libchallengeresponse.so shared library. Its not the goal to know what this function do. But we have to write a dumb one and compile it into the missed shared lib. But what is the signature of the function ? Lets get a look at the function call :

```bash
.text:08048A84                 mov     edx, off_804A054
.text:08048A8A                 mov     eax, off_804A050
.text:08048A8F                 mov     [esp+4], edx
.text:08048A93                 mov     [esp], eax
.text:08048A96                 call    _fillChallengeResponse
.text:08048A9B                 cmp     eax, 0FFFFFFFFh
.text:08048A9E                 jnz     short loc_8048AAC
.text:08048AA0                 mov     dword ptr [esp], 1 ; status
```

The fillChallengeResponse function take two (char*) arguments and return an int status. Lets write it :

```c
int fillChallengeResponse(char *a, char *b)
{
a[0] = 'A';
a[1] = 'B';
a[2] = 'C';
a[3] = 'D';
b[0] = '1';
b[1] = '2';
b[2] = '3';
b[3] = '4';
printf("fillChallengeResponse lib calledn");
return  0;
}
```

Then compile the  shared lib.

```bash
gcc -c -Wall -Werror -fpic libchallengeresponse.c
gcc -shared -o libchallengeresponse.so libchallengeresponse.o
cp libchallengeresponse.so /usr/lib/
ldconfig
```

Now running the binary gives :
fillChallengeResponse lib called
CSAW ChallengeResponseAuthenticationProtocol Flag Storage
It works fine. Lets analyze the binary. One interresting function is the sub_8048A01 function that will read the flag.txt file and print out the flag. But before that this function call another function sub_80487ED to check whether to print the flag or not. So the goal now is to set the ds:dword_804A0A0 var to 1. But how ? simply by giving the correct challenge response the the service.

The sub_8048825 reads 1byte from the stdin and then 4 cases are possible. If the char was 0xA0 -> the binary prints out the challenge. If the char was 0xE0 -> the binary will read 4 chars from the stdin buffer and will check if the challenge response is correct(as described below). If the char was 0x80-> the binary will check if the entred challenge response is correct and decide wether printing the flag or not. Any Other char -> The program will exist. Now lets look closer the the sub_8048825 function. It take as argument the entred char and apply the following operation to it.

```bash
.text:0804886D                 and     eax, 0Fh
.text:08048870                 mov     [ebp+var_11], al
.text:08048873                 mov     eax, off_804A050
.text:08048878                 movzx   edx, [ebp+var_11]
.text:0804887C                 shl     edx, 2
.text:0804887F                 add     eax, edx
.text:08048881                 mov     eax, [eax]
```

Now imagine that we entred the 0xA1 byte. (0xA1 & 0x0F) << 2 gives 4 The function in this case will print 4 chars from the **off_804A050+4** buffer. Have we some interrsting buffer arround off_804A050 ? Yes we have the **off_804A054** pointer that holds the challenge response. Now we can exploit the **sub_8048825** function to read its content. off_804A050 is a pointer the **unk_804A0C0** var which is a 32 bytes array. off_804A054 is a pointer to **unk_804A0E0** var which is a 32 bytes array. **unk_804A0C0** and **unk_804A0E0** are contiguous blocks.Now to read the challenge response value. **(32 >> 2) OR 0xaf = 0xA8** Putting it all together. 1) We send one byte **"xa8"** to the server. 2) The server sends us the challenge response(imagine that we got 4 bytes ABCD) 3) We send tback o the server 5 bytes **"xe0ABCD"** 4) The server cheks the challenge response which is correct and set **ds:dword_804A0A0** to 1 5) We send one byte **"x80"** 6) The server attempts to test about ds:dword_804A0A0 and the flag is sent. Now just redo this scenario 8 times to get the flag. The flag was :

The flag was : **flag{greetings_to_pure_digital}**
