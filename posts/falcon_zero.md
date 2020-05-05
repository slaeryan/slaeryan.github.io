# Introducing FalconZero - an undetectable, targeted implant for delivering second-stage payloads to the host machine

**Reading Time:** _16 minutes_

## Warning Ahead!
You could dominate the world if you read this post from top to bottom and follow all the instructions written here. Proceed at your own discretion, operator!

![Offensive Tools Meme](../assets/images/offensive-tools-meme.jpeg "Offensive Tools Meme")

## TL;DR
This tool is [![ForTheBadge built-with-love](http://ForTheBadge.com/images/badges/built-with-love.svg)](https://GitHub.com/Naereen/) for red team operators and offensive security researchers and ergo, being made open source and free(as in free beer).

It is available here: [https://github.com/slaeryan/FALCONSTRIKE](https://github.com/slaeryan/FALCONSTRIKE)

## Introduction
Ever since I completed my SLAE, I am completely enchanted by the power of shellcode. This feeling was only augmented when I heard a podcast by the wonderful guys at FireEye's Mandiant Red Team where they advocated the usage of shellcode in red teaming engagements for it's flexibility and it's ability to evade AV/EDRs among other things.

That's when I decided to play around with various shellcode injection techniques. Along the way, I thought of a ***cool*** technique and made an implant based on it that could deliver Stage-2 payloads to the target machine in a stealthy manner. 
But why stop there? Why not add some neat features to it and create a framework to aid red teamers to generate these implants as quickly and cleanly as possible.

That was the inception of the _FALCONSTRIKE_ project and _FalconZero_ is the first public release version Loader/Dropper of the _FALCONSTRIKE_ project. It implements the **BYOL(Bring Your Own Land)** approach as opposed to **LotL(Living off the Land)**.
But it's not your standard run-off-the-mill shellcode loader(more on this later).

<p align="center">
  <img src="../assets/images/FALCONSTRIKE.png">
</p>

You may think of _FalconZero_ as a loading deck for malware. In other words, _FalconZero_ is comparable to an undetectable gun that will fire a bullet(payload) on the host machine.

This is the reason it may not be classified as a malware per-se but rather a facilitator of sorts that helps the malware get undetected on the host.

## But there's plenty of tools that already do that. So what makes _FalconZero_ special?
While there are many excellent existing projects, this is not designed to be a replacement for them.

This is designed to be unique in its own way and there are quite a few of those features. So let's discuss them one by one.
### Separation of the final-stage payload from the implant
Some of the ways of storing the Stage-2 payload(shellcode) in the Stage-1 payload(Dropper) are:

1. Storing shellcode in .text section of Dropper
1. Storing shellcode in .data section of Dropper
1. Storing shellcode in .rsrc section of Dropper etc.

While these techniques remain quite popular but keeping both the shellcode and Dropper bundled together(even if its encrypted) is probably not a good idea from an OPSEC perspective.

Why so?

Imagine if the blue-teams get hold of an undetonated implant, not only will the Dropper get compromised but also the Stage-2 payload which can't be any good.

So it's best practise to separate the Dropper and the shellcode over network. In other words, the Dropper can connect to a remote server where the shellcode is hosted, fetch it from over there, prep it and then proceed to inject it into a host process which is exactly what has been implemented.
### Usage of Github for fetching the Stage-2 payload
Yep! You read that correctly. Github is used as the place where the payloads are stored. 
The implant connects to the appropriate Github repository and fetches the payload from there.

Why such a choice?

Simply because [Github](https://github.com) is largely considered a legitimate website and web traffic to Github will **not be considered malicious by network security products** and will probably not even be blocked in most organisations/offices as compared to using some attacker-owned web server hosting a payload which could be noisy as hell.

As a bonus point, this would save the operator time and money too ;)










<script id="asciicast-xGZ7B6Vn2byMWniewydzQCEco" src="https://asciinema.org/a/xGZ7B6Vn2byMWniewydzQCEco.js" async></script>

[VT Scan Results](https://www.virustotal.com/gui/file/987505a6c969112378bd074b43fb474710ad1d50c07c96a3b9dfb87e7f94a2c8/detection)

[Intezer Analyze Results](https://analyze.intezer.com/#/analyses/32930dbf-0bb4-4817-a682-75b3e87bbddb)
