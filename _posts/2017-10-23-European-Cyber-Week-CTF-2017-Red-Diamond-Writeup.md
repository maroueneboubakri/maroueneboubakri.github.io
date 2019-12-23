
Hi,

European Cyber Week CTF is a contest exclusively reserved for "European" students.

I'am not eligible to play but I joined the CTF later in order to keep practicing and learn new stuff .

Red Diamond in the Binary category was the hardest task regarding the number of players who solved it (7 solves).

The given file was a Windows binary for AMD64 !

I imported the binary file into IDAPro, then, with a top-down look at the string list I saw many paths to  **mruby** header files such as _/home/FlL/mruby1.3.0/mruby/include/mruby/value.h_

I "googled" mruby. It is a lightweight implementation of the Ruby language. From this point I figured out that my mission will be hard because I never wrote a single line of ruby code. But remember that my goal is to learn new stuff ! So ! Let's do it !

Mruby can be linked and embedded within an application, which is the case. In addition, Ruby programs can be compiled into byte code using the mruby compiler.

Typically the challenge file is an application embedding a mruby bytecode and an interpreter.

I spent some time reading about MRB file format and **RiteVM**.

MRB file starts with  _RITE00_... as magic number. I searched that pattern inside the binary and, yes !, I found it.

The bytecode program starts from offset 0x80A20 in the file (0x100482020 offset in memory)

I dumped the bytecode to a  _red_diamond.mrb_  file.

Mruby package provides an interpreter program "mruby". I downloaded mruby sources and I compiled it.

<!--more-->

```bash
root@maro-vm:~/ecw# git clone https://github.com/mruby/mruby.git
root@maro-vm:~/ecw# cd mruby/
root@maro-vm:~/ecw/mruby# ./minirake
```

  
I tried to run the binary file.

  

```bash
root@maro-vm:~/ecw/mruby# ./bin/mruby -b red_diamond.mrb
CRACKME!
NoMethodError: undefined method 'usleep' for main
```

  

Well ! I got a message printed to screen, however the program crashed due to call to undefined function  _usleep_.

  

I added the sleep module from here  _[https://github.com/matsumotory/mruby-sleep](https://github.com/matsumotory/mruby-sleep)_  ! But instead of specifying the github path, I cloned the module source and specified its path in the disk to  **build_config.rb**. I did this in order to be able to easily alter the code if needed.

  

``` bash
root@maro-vm:~/ecw/mruby# git clone https://github.com/matsumotory/mruby-sleep.git
root@maro-vm:~/ecw/mruby# nano build_config.rb
MRuby::Build.new do |conf|
   conf.gem 'mruby-sleep'
...
```

I did the same thing for the other requested modules. Those are,  **mruby-eval, mruby-iconv, mruby-md5 and mruby-pack.**


```bash
MRuby::Build.new do |conf|
...
  conf.gem :core => 'mruby-eval'
  conf.gem 'mruby-sleep'
  conf.gem :git => 'https://github.com/mattn/mruby-iconv.git'
  conf.gem :git => 'https://github.com/mattn/mruby-md5.git'
  conf.gem 'mruby-pack'
...

```

Then I compiled again mruby.
```bash

root@maro-vm:~/ecw/mruby# ./minirake clean
root@maro-vm:~/ecw/mruby# ./minirake

```

I tried again to run the program.

```bash
root@maro-vm:~/ecw/mruby#./bin/mruby -b red_diamond.mrb
CRACKME!
Let me check if you deserve a flag ...
NO :(

```

  
Good ! works fine !

Using -v option it is possible to see VM codes of the program.


```bash
root@maro-vm:~/ecw/mruby#./bin/mruby -b -v red_diamond.mrb

mruby 1.3.0 (2017-7-4)
irep 0x555555873e30 nregs=7 nlocals=3 pools=11 syms=12 reps=2
      000 OP_LOADSELF   R3
      001 OP_STRING     R4      L(0)    ; "CRACKME!"
      002 OP_SEND       R3      :puts   1
      003 OP_LOADSELF   R3
      004 OP_LOADSELF   R4
      005 OP_LOADL      R5      L(1)    ; 400000
      006 OP_SEND       R4      :rand   1
      007 OP_SEND       R3      :usleep 1
      008 OP_LOADSELF   R3
      009 OP_STRING     R4      L(2)    ; "Let me check if you deserve a flag ..."
      010 OP_SEND       R3      :puts   1
      011 OP_LOADSELF   R3
      012 OP_LOADSELF   R4
      013 OP_LOADL      R5      L(3)    ; 600000
      014 OP_SEND       R4      :rand   1
      015 OP_LOADL      R5      L(4)    ; 2000000
      016 OP_ADD        R4      :+      1
      017 OP_SEND       R3      :usleep 1
      018 OP_ONERR      050
      019 OP_TCLASS     R3
      020 OP_LAMBDA     R4      I(+1)   method
      021 OP_METHOD     R3      :found?
      022 OP_OCLASS     R3
      023 OP_LOADNIL    R4
      024 OP_CLASS      R3      :String
      025 OP_EXEC       R3      I(+2)
      026 OP_LOADSELF   R3
      027 OP_SEND       R3      :found? 0
      028 OP_JMPNOT     R3      046
      029 OP_LOADSELF   R3
      030 OP_STRING     R4      L(5)    ; "YES :)"
      031 OP_SEND       R3      :puts   1
      032 OP_GETCONST   R3      :MD5
      033 OP_STRING     R4      L(6)    ; "\342\235\250\342\225\257\302\260\342\226\241\302\260\342\235\251\342\225\257\357\270\265\342\224\273\342\224\201\342\224\273"
      034 OP_GETGLOBAL  R5      :$salt
      035 OP_ADD        R4      :+      1
      036 OP_SEND       R3      :md5_hex        1
      037 OP_MOVE       R1      R3              ; R1:flag
      038 OP_LOADSELF   R3
      039 OP_STRING     R4      L(7)    ; "\tflag is: '"
      040 OP_MOVE       R5      R1              ; R1:flag
      041 OP_STRCAT     R4      R5
      042 OP_STRING     R5      L(8)    ; "'"
      043 OP_STRCAT     R4      R5
      044 OP_SEND       R3      :puts   1
      045 OP_JMP        049
      046 OP_LOADSELF   R3
      047 OP_STRING     R4      L(9)    ; "NO :("
      048 OP_SEND       R3      :puts   1
      049 OP_JMP        069
      050 OP_RESCUE     R3
      051 OP_GETCONST   R4      :Exception
      052 OP_RESCUE     R3      R4      cont
      053 OP_JMPIF      R4      055
      054 OP_JMP        068
      055 OP_MOVE       R2      R3              ; R2:e
      056 OP_LOADSELF   R3
      057 OP_STRING     R4      L(10)   ; "fail!"
      058 OP_SEND       R3      :puts   1
      059 OP_LOADSELF   R3
      060 OP_MOVE       R4      R2              ; R2:e
      061 OP_SEND       R4      :message        0
      062 OP_SEND       R3      :puts   1
      063 OP_LOADSELF   R3
      064 OP_MOVE       R4      R2              ; R2:e
      065 OP_SEND       R4      :backtrace      0
      066 OP_SEND       R3      :puts   1
      067 OP_JMP        070
      068 OP_RAISE      R3
      069 OP_POPERR     1
      070 OP_STOP

irep 0x5555558743e0 nregs=25 nlocals=5 pools=10 syms=19 reps=1
      000 OP_ENTER      0:0:0:0:0:0:0
      001 OP_STRING     R5      L(0)    ; "utf-0"
      002 OP_STRING     R6      L(1)    ; "utf-9"
      003 OP_RANGE      R5      R5      0
      004 OP_SEND       R5      :to_a   0
      005 OP_STRING     R6      L(2)    ; "utf-10"
      006 OP_STRING     R7      L(3)    ; "utf-30"
      007 OP_RANGE      R6      R6      0
      008 OP_SEND       R6      :to_a   0
      009 OP_ADD        R5      :+      1
      010 OP_MOVE       R2      R5              ; R2:"\302\265"
      011 OP_LOADSELF   R5
      012 OP_GETCONST   R6      :Iconv
      013 OP_MOVE       R7      R2              ; R2:"\302\265"
      014 OP_LOADI      R8      8
      015 OP_SEND       R7      :[]     1
      016 OP_MOVE       R8      R2              ; R2:"\302\265"
      017 OP_LOADI      R9      16
      018 OP_SEND       R8      :[]     1
      019 OP_LOADI      R9      254
      020 OP_LOADI      R10     255
      021 OP_LOADI      R11     0
      022 OP_LOADI      R12     65
      023 OP_LOADI      R13     0
      024 OP_LOADI      R14     82
      025 OP_LOADI      R15     0
      026 OP_LOADI      R16     71
      027 OP_LOADI      R17     0
      028 OP_LOADI      R18     86
      029 OP_LOADI      R19     0
      030 OP_LOADI      R20     91
      031 OP_LOADI      R21     0
      032 OP_LOADI      R22     50
      033 OP_LOADI      R23     0
      034 OP_LOADI      R24     93
      035 OP_ARRAY      R9      R9      16
      036 OP_STRING     R10     L(4)    ; "C*"
      037 OP_SEND       R9      :pack   1
      038 OP_SEND       R6      :conv   3
      039 OP_SEND       R5      :eval   1
      040 OP_MOVE       R3      R5              ; R3:"\302\244"
      041 OP_JMPNOT     R5      043
      042 OP_JMP        045
      043 OP_LOADNIL    R5
      044 OP_RETURN     R5      normal
      045 OP_LOADT      R4                      ; R4:"\302\247"
      046 OP_MOVE       R5      R3              ; R3:"\302\244"
      047 OP_STRING     R6      L(4)    ; "C*"
      048 OP_SEND       R5      :unpack 1
      049 OP_MOVE       R2      R5              ; R2:"\302\265"
      050 OP_MOVE       R5      R3              ; R3:"\302\244"
      051 OP_LOADI      R6      0
      052 OP_LOADI      R7      7
      053 OP_RANGE      R6      R6      0
      054 OP_SEND       R5      :[]     1
      055 OP_MOVE       R3      R5              ; R3:"\302\244"
      056 OP_MOVE       R5      R4              ; R4:"\302\247"
      057 OP_JMPNOT     R5      062
      058 OP_MOVE       R5      R3              ; R3:"\302\244"
      059 OP_SEND       R5      :first  0
      060 OP_STRING     R6      L(5)    ; "W"
      061 OP_EQ         R5      :==     1
      062 OP_MOVE       R4      R5              ; R4:"\302\247"
      063 OP_JMPNOT     R5      068
      064 OP_MOVE       R5      R3              ; R3:"\302\244"
      065 OP_SEND       R5      :last   0
      066 OP_STRING     R6      L(6)    ; "a"
      067 OP_EQ         R5      :==     1
      068 OP_MOVE       R4      R5              ; R4:"\302\247"
      069 OP_JMPNOT     R5      077
      070 OP_MOVE       R5      R3              ; R3:"\302\244"
      071 OP_LOADI      R6      1
      072 OP_ADDI       R6      :+      1
      073 OP_SEND       R5      :[]     1
      074 OP_MOVE       R6      R3              ; R3:"\302\244"
      075 OP_SEND       R6      :first  0
      076 OP_EQ         R5      :==     1
      077 OP_MOVE       R4      R5              ; R4:"\302\247"
      078 OP_JMPNOT     R5      085
      079 OP_MOVE       R5      R3              ; R3:"\302\244"
      080 OP_LOADI      R6      1
      081 OP_SEND       R5      :[]     1
      082 OP_LOADI      R6      0
      083 OP_SEND       R6      :to_s   0
      084 OP_EQ         R5      :==     1
      085 OP_MOVE       R4      R5              ; R4:"\302\247"
      086 OP_JMPNOT     R5      095
      087 OP_MOVE       R5      R3              ; R3:"\302\244"
      088 OP_LOADI      R6      3
      089 OP_SEND       R5      :[]     1
      090 OP_SEND       R5      :to_i   0
      091 OP_SUBI       R5      :-      1
      092 OP_LOADI      R6      2
      093 OP_ADDI       R6      :+      2
      094 OP_EQ         R5      :==     1
      095 OP_MOVE       R4      R5              ; R4:"\302\247"
      096 OP_JMPNOT     R5      103
      097 OP_MOVE       R5      R3              ; R3:"\302\244"
      098 OP_STRING     R6      L(7)    ; "4"
      099 OP_SEND       R6      :to_i   0
      100 OP_SEND       R5      :[]     1
      101 OP_STRING     R6      L(8)    ; "9"
      102 OP_EQ         R5      :==     1
      103 OP_MOVE       R4      R5              ; R4:"\302\247"
      104 OP_JMPNOT     R5      113
      105 OP_MOVE       R5      R3              ; R3:"\302\244"
      106 OP_LOADI      R6      5
      107 OP_LOADI      R7      -1
      108 OP_RANGE      R6      R6      0
      109 OP_SEND       R5      :[]     1
      110 OP_SEND       R5      :first  0
      111 OP_STRING     R6      L(9)    ; "("
      112 OP_EQ         R5      :==     1
      113 OP_MOVE       R4      R5              ; R4:"\302\247"
      114 OP_JMPNOT     R5      122
      115 OP_MOVE       R5      R3              ; R3:"\302\244"
      116 OP_LOADSYM    R6      :[]
      117 OP_LOADI      R7      -2
      118 OP_SEND       R5      :send   2
      119 OP_SEND       R5      :to_f   0
      120 OP_LOADI      R6      8
      121 OP_EQ         R5      :==     1
      122 OP_MOVE       R4      R5              ; R4:"\302\247"
      123 OP_LOADI      R5      8
      124 OP_LAMBDA     R6      I(+1)   block
      125 OP_SENDB      R5      :times  0
      126 OP_MOVE       R5      R4              ; R4:"\302\247"
      127 OP_JMPNOT     R5      132
      128 OP_MOVE       R5      R2              ; R2:"\302\265"
      129 OP_SEND       R5      :size   0
      130 OP_LOADI      R6      16
      131 OP_EQ         R5      :==     1
      132 OP_MOVE       R4      R5              ; R4:"\302\247"
      133 OP_SETGLOBAL  :$salt  R3              ; R3:"\302\244"
      134 OP_RETURN     R4      normal          ; R4:"\302\247"

irep 0x555555874a40 nregs=8 nlocals=3 pools=0 syms=6 reps=0
      000 OP_ENTER      1:0:0:0:0:0:0
      001 OP_GETUPVAR   R3      4       0
      002 OP_JMPNOT     R3      015
      003 OP_GETUPVAR   R3      2       0
      004 OP_MOVE       R4      R1              ; R1:n
      005 OP_SEND       R3      :[]     1
      006 OP_GETUPVAR   R4      2       0
      007 OP_MOVE       R5      R1              ; R1:n
      008 OP_SEND       R5      :-@     0
      009 OP_SUBI       R5      :-      1
      010 OP_SEND       R4      :[]     1
      011 OP_MOVE       R5      R1              ; R1:n
      012 OP_ADDI       R5      :+      1
      013 OP_SEND       R4      :^      1
      014 OP_EQ         R3      :==     1
      015 OP_SETUPVAR   R3      4       0
      016 OP_RETURN     R3      normal

irep 0x555555874b30 nregs=3 nlocals=1 pools=0 syms=2 reps=2
      000 OP_TCLASS     R1
      001 OP_LAMBDA     R2      I(+1)   method
      002 OP_METHOD     R1      :first
      003 OP_TCLASS     R1
      004 OP_LAMBDA     R2      I(+2)   method
      005 OP_METHOD     R1      :last
      006 OP_LOADSYM    R1      :last
      007 OP_RETURN     R0      normal

irep 0x555555874c20 nregs=5 nlocals=2 pools=0 syms=1 reps=0
      000 OP_ENTER      0:0:0:0:0:0:0
      001 OP_LOADSELF   R2
      002 OP_LOADI      R3      0
      003 OP_SEND       R2      :[]     1
      004 OP_RETURN     R2      normal

irep 0x555555874ce0 nregs=5 nlocals=2 pools=0 syms=1 reps=0
      000 OP_ENTER      0:0:0:0:0:0:0
      001 OP_LOADSELF   R2
      002 OP_LOADI      R3      -1
      003 OP_SEND       R2      :[]     1
      004 OP_RETURN     R2      normal

```

  

Sadly there is not decompiler for the bytecode. I spent to much time reading about RiteVM opcodes and play with the interactive mruby shell "**mirb**" to understand what the program does.

  

The program starts by printing "CRACKME!" to screen, sleeps a while (random seconds), prints "Let me check if you deserve a flag ...", sleeps a while again perform a test on found method result then prints "NO ! :(" and exits.

  

If the found method returns 1 then it loads the content of $salt global variable, append it to "\342\235\250\342\225\257\302\260\342\226\241\302\260\342\235\251\342\225\257\357\270\265\342\224\273\342\224\201\342\224\273", compute the md5 over it, hex encode it and prints flag is and the flag value.

  
If found returns 0, then "NO :(" is printed and the program exists.

The program author has extended the String class by adding 3 methods ,  **first**,  **last** and  **found**.


It is obvious that irep 0x555555874c20 is the implementation of "_first_" method since it returns the element with index -1 of an array.

irep 0x555555874ce0 is for "_last_" method because it returns the element with index -1 of an array.

irep 0x5555558743e0 is the implementation of found method.

It reads the second argument passed to the program using  _eval(ARGV[2])_  then the argument value is checked if it is the expected value or not.

It reads the first element using the defined first method and check if it is equal to "W", if it is the case, it loads the last character and check if it is "a" if so, further checks is done.

Then the third character (index incremented by 2) is tested if it is equal to the first character.

Next, the index is decremented by 1 to test if the first character is equal to "0"

The fourth character must be equal to "4" + 1 = "5"

The fifth character must be equal to "9"

The sixth character must be "("

The seventh character must be "8"

Hence the correct password is  **W0W59(8a**


Finally the :$salt global variable is set to eval(ARGV[2]) which is the second argument.

The function return 1 if the argument satisfies the test otherwise it returns 0.

The strange thing is that by feeding the correct input to the program it kept saying NO ! :(

And since I'm sure that what I did is correct ! I putted the flag in the requested format and I submitted it to the platform and BOM! it is accepted ! and that's why maybe players failed to solve it ! or maybe that the there is many solutions for this challenge also.

 
Flag was  **ECW{983b428e721bcfceabf6c77d9e819d8d}**

  
And yes the author was right !

```bash
In [5]: print "\342\235\250\342\225\257\302\260\342\226\241\302\260\342\235\251\342\225\257\357\270\265\342\224\273\342\224\201\342\224\273"
❨╯°□°❩╯︵┻━┻

```

  

![](https://imgflip.com/s/meme/Table-Flip-Guy.jpg)

  

Another way to solve this challenge could be dealing with it as a blackbox by counting the number of instruction executed for each input.

This could not work here since the characters are not checked in order. You need in this case to trace the program execution. After googling a while, I found an interesting module which is "**mruby-debug-example**" it hooks the VM function which fetches the opcodes and dump it.

  

The module can be downloaded from here  [https://github.com/masuidrive/mruby-debug-example](https://github.com/masuidrive/mruby-debug-example)


Add it to your build_config.rb and compile again using minirake.


Now by running the program you can get a full execution trace !

```bash
...
0x555555876eec: OP_LOADSELF     R3
0x555555876ef0: OP_LOADSELF     R4
0x555555876ef4: OP_LOADL        R5      L(1)
0x555555876ef8: OP_SEND R4      :rand   1
0x555555876efc: OP_SEND R3      :usleep 1
0x555555876f00: OP_LOADSELF     R3
0x555555876f04: OP_STRING       R4      "Let me check if you deserve a flag ..."
0x555555876f08: OP_SEND R3      :puts   1
0x5555555ebe6c: OP_ENTER        0:0:1:0:0:0:0
0x5555555ebe70: OP_LOADI        R3      0
0x5555555ebe74: OP_MOVE R6      R1
0x5555555ebe78: OP_SEND R6      :size   0
0x5555555ebe7c: OP_MOVE R4      R6
0x5555555ebe80: OP_JMP          021
0x5555555ebed4: OP_MOVE R6      R3
0x5555555ebed8: OP_MOVE R7      R4
0x5555555ebedc: OP_LT   R6      :<      1
0x5555555ebee0: OP_JMPIF        R6      -23
0x5555555ebe84: OP_MOVE R6      R1
0x5555555ebe88: OP_MOVE R7      R3
0x5555555ebe8c: OP_SEND R6      :[]     1
0x5555555ebe90: OP_SEND R6      :to_s   0
0x5555555ebe94: OP_MOVE R5      R6
0x5555555ebe98: OP_LOADSELF     R6
0x5555555ebe9c: OP_MOVE R7      R5
0x5555555ebea0: OP_SEND R6      :__printstr__   1
Let me check if you deserve a flag ...0x5555555ebea4: OP_MOVE   R6      R5
0x5555555ebea8: OP_LOADI        R7      -1
0x5555555ebeac: OP_SEND R6      :[]     1
0x5555555ebeb0: OP_STRING       R7      "\n"
0x5555555ebeb4: OP_SEND R6      :!=     1
0x5555555ebeb8: OP_JMPNOT       R6      004
0x5555555ebebc: OP_LOADSELF     R6
0x5555555ebec0: OP_STRING       R7      "\n"
0x5555555ebec4: OP_SEND R6      :__printstr__   1
...

```

The author maybe added the sleep methods to prevent bruteforce ! You can at this level modify mrb_sleep.c source file and modufy the usleep implementation and make it sleeps for 0 seconds for example.

```c

93:    if (mrb_fixnum_p(argv[0]) && mrb_fixnum(argv[0]) >= 0) {
94:        usleep(0); //usleep(mrb_fixnum(argv[0]));
95:    } else {
96:        mrb_raise(mrb, E_ARGUMENT_ERROR, "time interval must be positive");
97:    }

```

You can than write a sample python script to bruteforce the password !

Here is a python script which can be used to find the first character.

```python
import subprocess

for i in range(32,127):
    s = chr(i)+"A"*7
    p = subprocess.Popen("./bin/mruby -b /root/red_diamond.mrb 0 0 '"+s+"'", shell=True, stdou$
    print "[+] %c -> %d"%(chr(i), len(p.stdout.readlines()))
```

```bash
...
[+] L -> 1821
[+] M -> 1821
[+] N -> 1821
[+] O -> 1821
[+] P -> 1821
[+] Q -> 1821
[+] R -> 1821
[+] S -> 1821
[+] T -> 1821
[+] U -> 1821
[+] V -> 1821
[+] W -> 1830
[+] X -> 1821
[+] Y -> 1821
[+] Z -> 1821
[+] [ -> 1821
[+] \ -> 1821
[+] ] -> 1821
[+] ^ -> 1821
[+] _ -> 1821
...

```

  

Found ! the first character is W.

  

But keep in mind that this way is good to quickly solve a CTF task but in general it is much leet to understand how things work (VM implementation etc..) !


That's all!
