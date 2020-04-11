# SLAE Exam Assignment 5 - Analyzing MSFVenom payloads

## Prologue
The fifth assignment for our SLAE certification exam is analyzing three shellcodes from the Metasploit MSFVenom tool and writing a suitable report for the same.

Almost all of us have used the Metasploit framework's `msfvenom` tool at some point and generated some payloads and used it somewhere but have you ever thought how does it function? How is it made?

If your answer is no then this is your chance to peek a glimpse at everything that happens under the hood of this powerful tool when you generate a payload.

For the purposes of this blog post I have chosen three payloads namely:
1. linux/x86/exec
1. linux/x86/shell_bind_tcp
1. linux/x86/shell_reverse_tcp

So without any further ado, here it it goes.

## Analyzing linux/x86/exec shellcode
As might be evident from the name, this is an execve shellcode and you can specify what to execute using the `CMD=` flag. Let's go ahead with `/bin/sh`.

We can generate the shellcode C-string using this command:
```
msfvenom -p linux/x86/exec CMD=/bin/sh --arch x86 -f c
```

The generated shellcode is as follows:
```
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 43 bytes
Final size of c file: 205 bytes
unsigned char buf[] = 
"\x6a\x0b\x58\x99\x52\x66\x68\x2d\x63\x89\xe7\x68\x2f\x73\x68"
"\x00\x68\x2f\x62\x69\x6e\x89\xe3\x52\xe8\x08\x00\x00\x00\x2f"
"\x62\x69\x6e\x2f\x73\x68\x00\x57\x53\x89\xe1\xcd\x80";
```

But this is basically an array of opcodes and doesn't make much sense does it eh?

Honestly it'd be good if we could look at the NASM source or better yet a visual representation of the flow right?

Fortunately enough for us there's a powerful tool known as `libemu` which allows us to do the same and more.

Assuming you have already installed the tool, let's look at how to generate a graph:
```
msfvenom -p linux/x86/exec CMD=/bin/sh --arch x86 -f raw | sctest -vvv -Ss 100000 -G linux-x86-exec.dot

dot linux-x86-exec.dot -Tpng -o linux-x86-exec.png
```

Let me paste the image here to aid in the analysis:

![linux-x86-exec](../assets/images/linux-x86-exec.png "linux-x86-exec")

Aaah! Looks pretty familiar now doesn't it? Some of you might have even analyzed it already as it's quite simple now but let's do a short analysis anyway.

First, we push 0x0b or 11 in decimal which is the syscall number for execve() into the stack and pop it into EAX which should contain the syscall number remember? Then we clear EDX register using `cwd` and push it into the stack. After that we proceed with setting up the arguments for execve() which in this case is "/bin/sh"(0x68732f = **hs/** & 0x6e69622f = **nib/**). Note that `/bin/sh` is reversed due to the little-endianess of the x86 architecture and also note that the usage of **-c** flag(0x632d) which actually reads the command from the _command_string_ rather than STDIN. Finally, we execute the execve syscall using the good old software interrupt _80h_.

## Analyzing linux/x86/shell_bind_tcp shellcode
This is a standard bind TCP shell payload(stageless) from Metasploit.

We can generate the payload using the following command:
```
msfvenom -p linux/x86/shell_bind_tcp LPORT=8080 -f c
```
And by piping the raw payload to the `sctest` tool, we can also generate a graph as follows:
```
msfvenom -p linux/x86/shell_bind_tcp LPORT=8080 -f raw | sctest -vvv -Ss 100000 -G linux-x86-bindshell.dot

dot linux-x86-bindshell.dot -Tpng -o linux-x86-bindshell.png
```

Let me include the image here to aid in the analysis:

![linux-x86-bindshell](../assets/images/linux-x86-bindshell.png "linux-x86-bindshell")

Whoa! This looks bigger than I expected:( Don't get disheartened, let's start with identifying the syscalls first, rest will be taken care of automatically. 

The syscalls are:
1. socket()
2. bind()
3. listen()
4. accept()
5. dup2()
6. execve()

Brilliant! Correlating with [Blog Post 1](https://slaeryan.github.io/posts/slae-assignment1-blogpost.html) where we created our own Bind TCP shellcode, we can see that the syscalls match in the exact order. Let's move in now for a deeper analysis.

### socket syscall
We can see that EBX is cleared using XORing with itself. Then we clear `EAX` using `mul` instruction(clever trick to save bytes!) and set up the stack for the socket syscall arguments:

1. domain - AF_INET = 0x02 for IPv4 addressing schema
2. type - SOCK_STREAM = 0x01 for a full-duplex byte stream socket communication
3. protocol - default 0x00 for single protocol support

After that we load the syscall number for socket = 0x167 in the lower part of EAX and execute the syscall.
### bind syscall
This is the step where we push into the stack for the sockaddr_in struct arguments with:

1. The server IP address - 0x00(0.0.0.0) to enable listening on all interfaces
2. The addressing schema - 0x02 for IPv4 addressing schema
3. The listening port - 0x901f(LPORT=8080, little-endian)

Now we set the stack for the bind syscall arguments with the the socket file descriptor created in the first syscall, socklen_t addrlen = 16 and the previously created sockaddr_in struct in a reverse-order(little endianess!) and load the appropiate syscall number in EAX before finally executing the syscall.
### listen syscall
This step should be easy to comprehend. Two things we should note here is that how they load into `EAX` the sockfd by directly referencing a location on the stack where it's located and loading of 0x04(SYS_LISTEN) into the lower part of EBX before loading the appropiate syscall number in EAX and executing the syscall
### accept syscall
In this step we just increment `EBX` by 1 making it 0x05 which is equal to SYS_ACCEPT. Then as usual we load the syscall value in EAX and execute the syscall.
### dup2 syscall
