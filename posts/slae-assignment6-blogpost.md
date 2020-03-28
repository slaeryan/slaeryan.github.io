## SLAE Exam Assignment 6 - Creating polymorphic shellcode

## Prologue
The second last assignment of the SLAE certification exam was quite an intriguing one. We were told to take three shellcodes of our choosing from [shell-storm](https://www.shell-storm.org/shellcode/) or [exploit-db](https://www.exploit-db.com/shellcodes) and create Polymorphic versions of those to evade _static signature-based_ AV/EDRs. Needless to say, it was a quite fun and brain-racking challenge.

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



