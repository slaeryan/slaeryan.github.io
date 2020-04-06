## SLAE Exam Assignment 3 - Creating an Egg-hunter shellcode

## Prologue
The third assignment for our SLAE Certification exam is creating an egg-hunter shellcode in such a way so as to make it's secondary payload configurable.

Egg-hunter remains an extremely important topic for study by exploitation analysts and rightfully so. 

So without any further ado, let's get down with it!

## So what is an egg-hunter anyway?
Imagine that you have found a buffer overflow exploit in a vulnerable software running in a remote machine but the buffer size or the space controlled by you is not nearly enough to fit a Bind/Reverse TCP shellcode :(

What do you do then?

Enter the Egg-hunter.

![Eggs](../assets/images/eggs.jpg "Eggs")

As many before me and also the ones who will be attempting this exam after me, there is one paper that stands out of all and is referenced by almost everyone perhaps because of it's simplicity and brilliance that is: [Safely Searching Process Virtual Address Space](https://github.com/upayansaha/SLAE-Code-Repository/blob/master/Assignment%203/egghunt-shellcode.pdf) by Skape.

I would recommend all the readers of this blog to read the paper first and then come back here.

I shall not discuss the paper in-depth in this blog-post. However, I shall mention some key concepts required for the analysis and implementation of an egg-hunter payload.
### Egg-hunter payload explained in a sentence
The egg-hunter is a shellcode usually optimized and small in size which is designed to find the final stage larger shellcode marked by an "egg"(4-8 bytes of data) and redirect execution flow to it once it finds it in the virtual address space.
### But how does it work?
The way the egg-hunter works is by iterating through the pages of memory and looking for the "egg" until its found.

Okay okay I get it but as far as I knew when a program tries to read unallocated memory in Linux, the program will crash throwing a `SIGSEGV` right?

Here comes Skape's out-of-the-box solution. He has devised a way to use the `access()` syscall to check whether a page of memory is accessible or not by using the memory address as argument and by checking the error code returned by the syscall, we can determine whether the page is accessible or not. This enables us to scan the memory safely.

Needless to say, if the page is inaccessible then it should skip to the next page directly otherwise it should continue scanning the page looking for our "egg".
### One final note
One last important matter to discuss is what should we consider as this "egg" that we are talking about?

Some implementations of the egg-hunter dictate the condition that the egg itself must be an executable for the egg-hunter to transfer the execution flow to the secondary payload stored somewhere else in memory.

Also, a single instance(4 bytes) of the egg shall be present in the egg-hunter itself so as to enable it to find it in memory by comparison right? Now there might be a scenario where the egg-hunter accidentally finds itself in process's memory thinking it has found the egg when it actually hasn't. So we must keep the egg as something unique.

In light of these issues, Skape has devised an 8-byte executable egg by simply appending the egg to itself making it unique and easily identifiable in the memory. While this is not strictly mandatory but in order to avoid adding unnecessary overheads and complexities, we shall also be using the original "egg" for the purposes of this blog post.

## Egg-hunter Assembly code
