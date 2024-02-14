---
title: Protostar - Stack 2
date: 2024-02-14 15:10:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack2]
---


## Conquering Stack2 at Protostar Machine

Hello ! Today we will continue our journey on [Protostar](https://exploit.education/protostar) machine  to solve  [Stack2](https://exploit.education/protostar/stack-two/) binary.

This challenge requiring us to perform a stack overflow and override the next value in the buffer with specific value as we see later.

### **Challenge Overview**
- **Challenge Name:** [Stack2](https://exploit.education/protostar/stack-two/)
- **Machine:** [Protostar](https://exploit.education/protostar)


### Step 1: Explore Binary
our first step always will be just exploreing binary , checking for input,output operations

![Stack2 Explore](/assets/img/protostar/stack2-explore.png)


### Step 2: Analysis

```sh
objdump -M intel -d /opt/protostar/bin/stack2 | grep '<main>' -A 33
```

![Stack2 Explore](/assets/img/protostar/stack2-analysis.png)

Analysis of `main` Function in `stack2` :

1. Function Prologue:
   - `push ebp`: save the base pointer.
   - `mov ebp, esp`: set up a new base pointer.
   - `and esp, 0xfffffff0`: align the stack.
   - `sub esp, 0x60`: allocate space for local variables.

2. Function Call to `getenv`:
   - `mov DWORD PTR [esp], 0x80485e0`: set the argument for `getenv` which as we see while exploreing the binary above , the environment variable called `GREENIE`.
   - `call 804837c <getenv@plt>`: call the `getenv` function and store the result in `eax`.
   - `mov DWORD PTR [esp+0x5c], eax`: store the result in the local variable at `[esp+0x5c]`.

3. Conditional Check:
   - `cmp DWORD PTR [esp+0x5c], 0x0`: compare the value at `[esp+0x5c]` with `0x0`.
   - `jne 80484c8 <main+0x34>`: jump to `0x80484c8` if the comparison is not equal.
   - this is interesting :) !
4. Branch on False:
   - If `jne` is not taken, it means the value at `[esp+0x5c]` is zero.
   - `mov DWORD PTR [esp+0x4], 0x80485e8`: set the argument for `errx` which is address for string error message.
   - `call 80483bc <errx@plt>`: call the `errx` function and exit.
  - 
5. Memory Operations:
   - `mov eax, DWORD PTR [esp+0x5c]`: move the value at `[esp+0x5c]` to `eax`.
   - `mov DWORD PTR [esp+0x4], eax`: move the value of `eax` to `[esp+0x4]`.
   - `lea eax, [esp+0x18]`: get the effective address of `[esp+0x18]` and store it in `eax`.
   - `mov DWORD PTR [esp], eax`: move the value of `eax` to `[esp]`. so the 2 process above prepare src and dst argument required for `strcpy` function call 
   - `call 804839c <strcpy@plt>`: call the `strcpy` function.

6. Memory Comparison:
   - `cmp eax, 0xd0a0d0a`: compare the result of `strcpy` with `0xd0a0d0a`.
   - `jne 80484fd <main+0x69>`: jump to `0x80484fd` if the comparison is not equal.

7. Branch on True:
   - so the code will jne if [esp+0x58] doesn,t matches `0x0d0a0d0a` if `jne` is not taken, it means the value matches `0xd0a0d0a`.
   - mov DWORD PTR [esp], 0x8048618`: set the argument for `puts` which is an address for string on stack.
   - `call 80483cc <puts@plt>`: call the `puts` function.

8. Branch on False:
   -  as we say before if [esp+0x58] doesn,t matches `0x0d0a0d0a` then it will execute `jne` instruction
   - `mov edx, DWORD PTR [esp+0x58]`: move the value at `[esp+0x58]` to `edx`.
   - `mov eax, 0x8048641`: move the address of the format string to `eax`.
   - `mov DWORD PTR [esp+0x4], edx`: move the value of `edx` to `[esp+0x4]`.
   - `mov DWORD PTR [esp], eax`: move the value of `eax` to `[esp]`.
   - `call 80483ac <printf@plt>`: call the `printf` function.
9. Function Epilogue:
   - `leave`: restore the stack frame (equivalent to `mov esp, ebp` followed by `pop ebp`).
   - `ret`: return from the function.

1. during run it first call `getenv("GREENIE")` to check env `GREENIE` if provided and if not provided it call errx with a error message which print it and exit 
2. if env var `GREENIE` provided will be copied to buffer
3. check if `[esp+0x58]==0xd0a0d0a` then call puts,print then return and else call print only and return

looks like stack1 but instead of takeing arguments as input it take it as input variable and also the value it compare `esp+0x58c` are just different as it equal `0xd0a0d0a`

so it will be easy , exploitation pattern is look like [My Writeup for Protostar Stack1](/posts/protostar_stack1/)

### Step 3: Debugging
![Stack2 Debug](/assets/img/protostar/stack2-gdb-debug.png)


1. disassemble `main` and set breakpoint on `mov eax,DWORD PTR [esp+0x58]` 
3. run it with `GREENIE env` var assigned to our fuzz paylaod
4. determine which value `dword[esp+0x58]` hold

![Stack2 found offset](/assets/img/protostar/stack2-gdb-found-offset.png)

so we found it contain `0x51515151` which represent ascii`QQQQ`.. Great!

### Step 4: Craft Solution
Now Let,s Craft Solution
write to : `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP"
esp_plus_58h="\x0a\x0d\x0a\x0d" # represent 0xd0a0d0a on little endian machines
print(fuzz+esp_plus_58h)
```

now run it
![Stack2 Solved](/assets/img/protostar/stack2-solved.png)

and We Solve it ;) ! Happy Hacking <3
