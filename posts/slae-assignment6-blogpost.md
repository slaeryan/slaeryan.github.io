## SLAE Exam Assignment 6 - Creating polymorphic shellcode

## Prologue
The second last assignment of the SLAE certification exam was quite an intriguing one. We were told to take three shellcodes of our choosing from [shell-storm](https://www.shell-storm.org/shellcode/) or [exploit-db](https://www.exploit-db.com/shellcodes) and create Polymorphic versions of those to evade _static signature-based_ AV/EDRs. Needless to say, it was a quite fun and brain-racking challenge.

So without any further ado, let's get down with it!

## What is polymorphic shellcode?
It is impossible to explain polymorphic shellcode if we don't understand what is polymorphism.

Polymorphism comes from the Greek words: _poly_ meaning "many" and _morph_ meaning "forms". In other words it roughly translates to "many forms" of the same thing.

In the context of shellcodes, we mean to say that a 

