# SLAE Exam Assignment 5 - Analyzing MSFVenom payloads

## Prologue
The fifth assignment for our SLAE certification exam is analyzing three shellcodes from the Metasploit MSFVenom tool and writing a suitable report for the same.

Almost all of us have used the Metasploit framework's `msfvenom` tool at some point and generated some payloads and used it somewhere but have you ever thought how does it function? How is it made?

If your answer is no then this is your chance to peek a glimpse at everything that happens under the hood of this powerful tool when you generate a payload.

For the purposes of this blog post I have chosen three payloads namely:
1. linux/x86/exec CMD=/bin/sh
1. 
1.

So without any further ado, here it it goes.

## Analyzing linux/x86/exec CMD=/bin/sh
As might be evident from the name, this shellcode executes `/bin/sh` on the machine.

We can generate the shellcode C-string using this command:
```
msfvenom -p linux/x86/exec CMD=/bin/sh --arch x86 -f c
```

