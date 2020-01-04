---
title: Hackit CTF 2016 - L4bR4t - rev375 - writeup
---

Hello,

In this task we were given a shared library “_SecretLabXLib.so_” and a zip file containing an encrypted and a plain jpeg files.

**Task description:**

There was some photos of unknown experiment taken in a secret lab-X for they internal archive. After that the device from which the shot was made, immediately load crypto-trigger, whose function – to ensure the confidentiality of image data (this is exactly how it should be in a super-secret laboratories?).

It is known that, due to some floating code errors, the trigger has not completed his work and not all the photos was encrypted.

We managed to get a binary, which has something to do with that crypto-trigger software.

And now we have a good chance to find out what secrets hides laboratory-X.

The first step I did is printing the symbol table of the shared library to figure out what functions are exported.

**Analyzing the shared object**

By running  _objdump -T_  I got mangled symbols name. De-mangling can be done using _nm -C_.

<!--more-->

```c++
root@maro-vm:~/hackit/rev375# nm -C SecretLabXLib.so
...
0000000000001d0f T AddRoundKey(unsigned char (*) [4], unsigned int const*)
0000000000002b02 T MixColumns(unsigned char (*) [4])
00000000000029c5 T InvShiftRows(unsigned char (*) [4])
0000000000002419 T InvSubBytes(unsigned char (*) [4])
000000000000186e T ccm_format_assoc_data(unsigned char*, int*, unsigned char const*, int)
00000000000035c5 T InvMixColumns(unsigned char (*) [4])
000000000000174c T ccm_prepare_first_ctr_blk(unsigned char*, unsigned char const*, int, int)
0000000000001950 T ccm_format_payload_data(unsigned char*, int*, unsigned char const*, int)
00000000000019fb T SubWord(unsigned int)
00000000000017a9 T ccm_prepare_first_format_blk(unsigned char*, int, int, int, int, unsigned char const*, int)
00000000000014fd T xor_buf(unsigned char const*, unsigned char*, unsigned long)
0000000000001650 T SuperSecretLabCryptor2000::decrypt_cbc(unsigned char const*, unsigned long, unsigned char*, unsigned int const*, int, unsigned char const*)
0000000000001faa T SubBytes(unsigned char (*) [4])
0000000000002888 T ShiftRows(unsigned char (*) [4])
0000000000004fe0 r sbox
00000000000051e0 r gf_mul
00000000000050e0 r invsbox
0000000000004a48 T SuperSecretLabCryptor2000::decrypt(unsigned char const*, unsigned char*, unsigned int const*, int)
0000000000001554 T SuperSecretLabCryptor2000::encrypt_cbc(unsigned char const*, unsigned long, unsigned char*, unsigned int const*, int, unsigned char const*)
0000000000004498 T SuperSecretLabCryptor2000::encrypt(unsigned char const*, unsigned char*, unsigned int const*, int)
0000000000001284 T SuperSecretLabCryptor2000::init_iv()
00000000000011b0 T SuperSecretLabCryptor2000::SuperSecretLabCryptor2000()
0000000000001222 T SuperSecretLabCryptor2000::init_key()
00000000000012d0 T SuperSecretLabCryptor2000::CryptFile(char*)
0000000000001ad2 T SuperSecretLabCryptor2000::key_setup(unsigned char const*, unsigned int*, int)
00000000000011b0 T SuperSecretLabCryptor2000::SuperSecretLabCryptor2000()
U operator new[](unsigned long)@@GLIBCXX_3.4
```

_SuperSecretLabCryptor2000_ class exports all methods requested to perform encryption/decryption.

  

It is obvious that  **CBC** is used as mode of operation.

  

Still need to figure out what Cryptographic algorithm, Key length, IV and Key were used.

  

Before to go deeper in analysis, I would to like to mention at this level that the shared object is a white-box cryptography implementation.

  

How ?  _CryptFile_ method takes as argument only filename, no encryption key is specified. The key is instantiated at runtime.

  

**What is white-box cryptography?**

  

In few words, white-box cryptography is aimed at protecting secret keys from being disclosed in a software implementation.

  

The main idea is to rewrite a key-instantiated version so that all information related to the key is “hidden”.

  

More details in this paper: http://joye.site88.net/papers/Joy08whitebox.pdf

  

**What encryption algorithm is implemented ?**

  

By getting a look at substitution box (_SBox_) you can determine that it is related to  **AES**.

  
```c
unsigned char sbox[256] ={99, 124, 119, 123, 242, 107, 111, 197, 48...};
```


**What is the Key length ?**

  
Can be determined by looking at  _CryptFile(char*)_  method disassembly. The key length is passed as argument to key_setup method.

```assembly
...
mov     ecx, 100h       ; int
...
call    __ZN25SuperSecretLabCryptor20009key_setupEPKhPji
```
  

The key length is 256 bits.


**What are the initialization vector (IV) and the Key ?**

To make life easier for me I chose to use the library to extract the (IV, Key) rather than reversing the  _key_init_,  _iv_init_ and  _setup_key_ functions.

Following is general overview of the encryption process.

The constructor does the following operations:

-Initializes a timestamp attribute through  _time_ function.

-Initializes key attribute through  _init_key_ method. The timestamp attribute is used here to add randomness to resulted key.

-Initializes the IV attribute using  _init_iv_ method. IV depends on initialized key (xoring with first 16 bytes of key).

  
CryptFile method reads the provdied file and calls  _key_setup_ method.  _key_setup_ takes the initialized key and the key length as argument and generates the final key to be used in encryption.

The content encryption is done by invoking  _encrypt_cbc_ method which takes as argument, respectively, input buffer, buffer length, output buffer, key, key length and iv.

Finally the result is written back to the file.

To perform encryption, I wrote the following C code. It is based on the provided library. It simulates  _SuperSecretLabCryptor2000_ object creation and calls the  _CryptFile_ method.

I did it with C not with C++ ! This is ugly but for sure there is a way to import a C++ class from a shared object and use it with no header file provided.

The code also prints out the Key and IV.

**My Solver**

```c
#include <stdio.h>
#include <stdint.h>
#include <stdlib.h>
#include <dlfcn.h>
#include <time.h>
#include <string.h>

  int main(int argc, char * * argv) {
    void * handle;

    char * error;
    handle = dlopen("./SecretLabXLib.so", RTLD_LAZY);
    if (!handle) {
      fprintf(stderr, "%s\n", dlerror());
      exit(1);
    }
    dlerror();

    void * obj = malloc(64);
    memset(obj, 0, 64);

    //uint32_t ts = strtol(argv[1], NULL, 10);

    * (uint32_t * ) obj = 1466136077; /*time(0);*/ 
 * (uint64_t * )(obj +  4) = 0; 
 * (uint64_t * )(obj + 12) = 0; 
 * (uint64_t * )(obj + 20) = 0; 
 * (uint64_t * )(obj + 28) = 0; 
 * (uint64_t * )(obj + 36) = 0; 
 * (uint64_t * )(obj + 44) = 0;

    long( * init_key)(void * );
    init_key = dlsym(handle, "_ZN25SuperSecretLabCryptor20008init_keyEv");

    if ((error = dlerror()) != NULL) {
      fprintf(stderr, "%s\n", error);
      exit(1);
    }

    ( * init_key)(obj);

    long( * init_iv)(void * );
    init_iv = dlsym(handle, "_ZN25SuperSecretLabCryptor20007init_ivEv");

    if ((error = dlerror()) != NULL) {
      fprintf(stderr, "%s\n", error);
      exit(1);
    }

    ( * init_iv)(obj);

    //print Timestamp|Key|IV
    int i;
    for (i = 0; i < 52; i++)
      printf("\\x%02x", * ((char * )(obj + i)) & 0xff);
    printf("\n");

    //crypt_file
    int( * crypt)(void * , char * );
    crypt = dlsym(handle, "_ZN25SuperSecretLabCryptor20009CryptFileEPc");

    if ((error = dlerror()) != NULL) {
      fprintf(stderr, "%s\n", error);
      exit(1);
    }

    ( * crypt)(obj, "dat.2.jpg");

    printf("[+] Done\n");

    dlclose(handle);
    return 0;
  }
  ```

But stop ! how can you get the correct key to decrypt the pictures ? In other words, what is the value of the correct timestamp used to generate the correct key and iv ?

I assumed that the timestamp used to encrypt the files is simply the last modification time of the encrypted file !

By running:

```bash
root@maro-vm:~/hackit/rev375# stat -c %Y *

1469990552
1469990552
1475262516 (plain)

1469990552

1469990552
1469990552
1469990104 (plain)
```
  
All encrypted files have the same last modification time. Using 1469990552 in my C code does not give a valid (Key, IV) !

By opening a plain picture with a hex editor I noticed that there is Fri, 17 Jun 2016 04:01:17 as creation date in exif metadtas in addition to XCryptoPicture as software.

```bash
root@maro-vm:~/hackit/rev375# date -d "Fri, 17 Jun 2016 04:01:17" +"%s"
1466128877
```
  

Running the the code with  **1466128877** timestamp gave the same encrypted header as the other encrypted pictures. It is the good one !

  

The correct key and iv are printed also.

  
Finally put all together in one python script.

```python

from Crypto.Cipher import AES

key = "\x49\xa5\xe7\x43\xe7\xfb\x05\xef\x88\x4c\x8c\x52\x7b\x61\x9c\x7c\x6b\xe1\x83\xd1\x5b\x46\x97\x48\x50\x98\x65\xbc\x3f\xa7\x70\x68"

IV="\x22\x44\x64\x92\xbc\xbd\x92\xa7\xd8\xd4\xe9\xee\x44\xc6\xec\x14"
mode = AES.MODE_CBC

with open("dat.1.jpg", 'rb') as cimg:

with open("dat.1.p.jpg", 'wb') as pimg:

decryptor = AES.new(key, mode, IV=IV)
        img = decryptor.decrypt(cimg.read())
        pimg.write(img)
```

Now we are able to decrypt the jpeg files and see the flag.


flag:  **h4ck1t{CrYp70_3xxP3R1m3N75_VV0n7_4LVV4Y5_3ND_VV3ll}**


Cheers :D
