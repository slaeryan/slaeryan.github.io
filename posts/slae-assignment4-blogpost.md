# SLAE Exam Assignment 4 - Creating a custom shellcode encoder

## Prologue
The fourth assignment that we have for our SLAE Certification exam is creating a custom shellcode encoder using all that we have learnt so far from the course.

In this blog post I am going to discuss what is a shellcode encoder, walk you guys through the creation process and at last show you a demo of what we have created.

So without any further ado, here it goes!

## Shellcode Encoder Explained
So the big question is what's a shellcode encoder?

I'd define an encoder as a piece of code that takes your shellcode as input and encodes/obfuscates/morphs it in way such that all bad characters including null bytes, new lines etc are removed from the shellcode and also appends a small stub to the shellcode which aids in decoding/changing it back to the original shellcode for execution on the target system.

This has another obvious benefit apart from removing bad characters as many of you might know from practical experience.

It obfuscates the shellcode which may hide it's true intentions from some AV/EDR solutions therefore lowering the detection rate on some site like VirusTotal but make no mistake, **AV evasion is not the primary objective of using a shellcode encoder** and it provides no real security to your payload because there's **no encryption happening** which means the encoding is usually trivial to reverse.

Ergo, always use encryption for implant security, AV/EDR evasion and Encoders for a preliminary obfuscation and removing bad characters. Ideally, you would want to use both. Refer to [Blog Post 7](https://slaeryan.github.io/posts/slae-assignment7-blogpost.html) for a primer on creating your own shellcode crypter.

So with that being explained many of you might have used the famous(or rather infamous!)Shikata Ga Nai Encoder from Metasploit package to obfuscate or bypass some AVs till it got burnt out and now all payloads keep getting attributed :(

Worry not! Here's how you can create your own shellcode encoder from scratch.

## Creating an Encoder prototype