# Introducing MIDNIGHTTRAIN - A Covert Stage-3 Persistence Framework utilizing UEFI variables

**Reading Time:** _16 minutes_

## TL;DR
This tool is [![ForTheBadge built-with-love](http://ForTheBadge.com/images/badges/built-with-love.svg)](https://GitHub.com/Naereen/) for red team operators, offensive security researchers and defenders alike ergo, being made open source and free(as in free beer).

It is available here: [https://github.com/slaeryan/MIDNIGHTTRAIN](https://github.com/slaeryan/MIDNIGHTTRAIN)

Fair warning: This code has been tested in my lab extensively but bugs are to be expected. Furthermore, I'm not a professional coder so get ready to read crappy code! However, I have given it my best possible efforts  and if you find bugs/improvements in the code don't feel shy to contact me!

## Introduction
One of my favourite hobbies is to read APT reports and look for the TTPs(Tactics, Techniques and Procedures) used by apex adversaries(Read as: State-backed Threat Groups) and later attempt to recreate it or a variant of it myself in my lab. So last Friday was no different, I was glossing over a report by [ESET](https://www.welivesecurity.com/2019/11/21/deprimon-default-print-monitor-malicious-downloader/) of **DePriMon** which basically registered as the default port monitor to achieve persistence on the host(Hence the name). This _alleged CIA_ implant is the first reported case of [T1547.010 Boot or Logon Autostart Execution: Port Monitors](https://attack.mitre.org/techniques/T1547/010/) found in the wild and it automatically piqued my interest



