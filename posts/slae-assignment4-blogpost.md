# SLAE Exam Assignment 4 - Creating a custom shellcode encoder

## Prologue
The fourth assignment that we have for our SLAE Certification exam is creating a custom shellcode encoder using all that we have learnt so far from the course.

In this blog post I am going to discuss what is a shellcode encoder, walk you guys through the creation process and at last show you a demo of what we have created.

So without any further ado, here it goes!

## Shellcode Encoder Explained
So the big question is what's a shellcode encoder?

I'd define an encoder as a piece of code that takes your shellcode as input and encodes/morphs it in way such that all bad characters including null bytes, new lines etc are removed from the shellcode and also appends a small stub to the shellcode which aids in decoding/changing it back to the original shellcode for execution on the target system.

This has another obvious benefit apart from removing bad characters.

It obfuscates the shellcode which may hide it's true intentions from some AV/EDR solutions therefore lowering the detection rate.