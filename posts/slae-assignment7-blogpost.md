# SLAE Exam Assignment 7 - Creating a custom shellcode crypter

## Prologue
After an absolutely amazing journey that was the x86 Assembly Language and Shellcoding on Linux course brought to us by our instructor Vivek Ramachandran and Pentester Academy, this is the last but definetely not the least enjoyable assignment that we had to do for our SLAE certification i.e. create a custom shellcode crypter that will allow us to bypass AV/EDR signatures.

For this assignment, we could use any programming language of our choice and I chose **Python3**(Hey! why not?) simply beacause it's a breeze to create prototypes in Python before finally implementing it in some language like C/C++. 

So what are we waiting for? Let's get down with it!

## But let's talk about the implementation details first!
So we are going to be using a cryptographic library known as NaCl which is an abbreviation for is an abbreviation for "Networking and Cryptography library" as the principal encryption library.

Fortunately for us, since this library was created by a very well-known mathematician and cryptographer known as D.J. Bernstein, we do not need to worry about the security or the mathematical backend of this library and we can move on directly with the practical aspects of it.
