---
title: 3DS CTF 2016 - High Power Computing - rev500 - writeup
---

Hello,

We were given a program which is pretended to print the flag by the end of its execution. The first run prints that it will takes days to finish the execution. The question to answer is, what makes it take this too much time ? The quick answer is simple operations such as multiplication, division, modulo,exponentiation...are made using loops. The program has several functions such as  _um()_,_feliz()_,_natal()_,  _ano()_... each function is responsible for a specific operation.

The job is simply to analyse function by function, figure out what it does and simplify it using arithmetic operators or optimized functions.

For example let's start with  _um()_  function

<!--more-->

```c
__uint64 um(__uint64 a1)
{
  return a1 - 42 + 43;
}
```

The function is just incrementing a variable. Can be just:

```c
__uint64 um(__uint64 a1)
{
  return a1 + 1;
}
```

  

Now what functions call  _um()_  ? Let's take _natal()_

```c
 __uint64 natal(__uint64 a1,  __uint64 a2)
{
   __uint64 j;
   __uint64 i;
   __uint64 v5;

  v5 = 0LL;
  for ( i = 0LL; i < a1; ++i )
    v5 = um(v5); // -> v5 + = 1;
  for ( j = 0LL; j < a2; ++j )
    v5 = um(v5); // -> v5 + = 1;
  return v5;
}
```

The function is just adding 2 numbers, and can be simplified to:

```c
 __uint64 natal(__uint64 a1,  __uint64 a2)
{
return a1 + a2;
}
```


We do the same, What functions use  _natal()_  ? Let's take _novo()_

```c
 __uint64 novo( __uint64 a1,  __uint64 a2)
{
   __uint64 i;
   __uint64 v4;

  v4 = 0LL;
  for ( i = 0LL; i < a1; ++i )
    v4 = natal(v4, a2); // -> v4 = v4 + a2
  return v4;
}
```

The function is multiplying 2 numbers and Can be only:

```c
 __uint64 novo( __uint64 a1,  __uint64 a2)
{
return a1 * a2;
}
```

 
And so on, until I got the following simplified version of the program:

```c
#include<stdio.h>
typedef unsigned long __uint64;
__uint64 modpow(__uint64 a, __uint64 b) {
//Source : https://github.com/pantaloons/RSA/blob/master/single.c
 __uint64 res = 1;
 while(b > 0) {
  if(b & 1) {
   res = (res * a) /*%c*/;
  }
  b = b >> 1;
  a = (a * a) /*%c*/;
 }
 return res;
}
int  main(int argc, const char **argv, const char **envp)
{
  __uint64 v3;
  __uint64 v4;
  __uint64 v5;
  __uint64 v6;
  __uint64 i;
  __uint64 v9;
  v9 = 0LL;
  for ( i = 0LL; i <= 0xC0DE41; ++i )
  {
    v3 = i / 0x29uLL;
    v4 = modpow(3uLL,v3);
    v5 = v4 + v9;
    v6 = v5 * i;
    v9 = v6 % 0x41DEADBABEC0FFEEuLL;
  }
  printf(
   "%04llx%04llx%04llx%04llx\n",
    (__uint64)v9 >> 48,
    (v9 & 0xFFFF00000000uLL) >> 32,
    (__uint64)((unsigned int)v9 & 0xFFFF0000) >> 16,
    (unsigned short)v9,
    i);
  return 0;
}
```

Run and compile prints the value 39a667347306e209 and the flag was  **3DS{39a667347306e209}**

  
That's all :)
