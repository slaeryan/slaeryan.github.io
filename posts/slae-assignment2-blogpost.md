# SLAE Exam Assignment 2 - Creating a Reverse TCP shellcode

## Prologue
So the second task that we have for our SLAE certification exam is creating a reverse TCP shellcode for Linux/x86 architecture using the knowledge that we have gathered from the wonderful course and the shellcoding techniques that we have accumulated over time.

To be honest this was easily one of my most favourite assignments that I had the pleasure of doing.

Why so? Let me tell you why.

So we create Meterpreter reverse TCP payloads all the time for exploitation purposes right? But invariably what happens is our payload getting caught and detected by AVs and EDRs. But that's all history now.
No more getting caught by AVs and EDRs and next-gen security products because now we can create our very own custom reverse TCP shellcode that is capable of bypassing all of the aforementioned security solutions!
Also there's no ignoring the fact that we hacker tribes get a different joy from using our own custom tools wherever possible in red-teaming engagements/CTFs yada yada yada.

So without any further ado let's get down with it!

## So what is Reverse TCP anyway?
Let's look at a visual representation first and then we'll talk about it.

![Reverse TCP Overview](../assets/images/reverse_tcp_overview.png "Reverse TCP Overview")

We can clearly see from this image that in case of a Reverse TCP shell, the client a.k.a. the victim/target machine beacons out to the server a.k.a. the attacker machine who is listening for connections at a specified port. Now after the server receives a connection, it can then send out commands to the client for it to execute on the victim machine.
