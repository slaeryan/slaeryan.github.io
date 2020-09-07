# Introducing MIDNIGHTTRAIN - A Covert Stage-3 Persistence Framework utilizing UEFI variables

**Reading Time:** _16 minutes_

## TL;DR
This tool is [![ForTheBadge built-with-love](http://ForTheBadge.com/images/badges/built-with-love.svg)](https://GitHub.com/Naereen/) for red team operators, offensive security researchers and defenders alike ergo, being made open source and free(as in free beer).

It is available here: [https://github.com/slaeryan/MIDNIGHTTRAIN](https://github.com/slaeryan/MIDNIGHTTRAIN)

Fair warning: This has been made as a small weekend project and while the code has been tested in my lab extensively, bugs are to be expected. Furthermore, I'm not a professional coder so get ready to read crappy code! However, I have given it my best possible efforts and if you find bugs/improvements in the code don't feel shy to contact me!

## Introduction
One of my favourite hobbies is to read APT reports and look for the TTPs(Tactics, Techniques and Procedures) used by apex adversaries(Read as: State-backed Threat Groups) and later attempt to recreate it or a variant of it myself in my lab.

So last Friday was no different, except that this time I was actually going through the CIA Vault7 leaks specifically the EDB branch documents when suddenly [this](https://wikileaks.org/ciav7p1/cms/page_26968084.html) came to my attention which describes the theory behind NVRAM variables.
Immediately it piqued my interest and I thought to myself well what can you do with it and what are the possible implications?

Turns out, these variables are not only accessible from the user-mode but also it's an awesome place to hide your shit like implants, config data, stolen goodies and whatnot which I found out after watching an enlightening [DEFCON talk](https://youtu.be/q2KUufrjoRo), thanks to Topher Timzen and Michael Leibowitz.

In the talk, they do give a demo using C# but the attendees are encouraged to figure out their own way to weaponize this technique.

So it got me thinking of various ways to weaponize this and suddenly I remembered glossing over a report by [ESET](https://www.welivesecurity.com/2019/11/21/deprimon-default-print-monitor-malicious-downloader/) some time back describing an _alleged CIA_ implant(Ironically again!) named **DePriMon** which registered as the default print monitor to achieve persistence on the host(hence the name).

That was the birth of the **MIDNIGHTTRAIN** framework. Over the next two days, I spent time coding it and then a couple of days more for writing this post.

## Of NVRAM variables, Print Monitors, Execution Guardrails with DPAPI etc. oh my my!
For the uninitiated readers, don't be scared of these buzzwords for I can guarantee you that this is absolutely nothing to be scared of and I shall explain each of these individual components(and the motivation behind it) one by one.

### NVRAM Variables
Just know that these are variables used by UEFI to store data that persist between boots. Needless to say, this data will survive a full reinstallation of the Operating System.

Sound like a stealthy place to hide your life's secrets yet?

What's more? As easy as it is to write data into firmware variables from user-mode, it is incredibly difficult(if not downright impossible) for the defenders to enumerate the data from the same.

Now, conviniently for us attackers, Microsoft provides us with fully-documented API access to the magical land of firmware variables using:

1. [SetFirmwareEnvironmentVariable()](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-setfirmwareenvironmentvariablea) - To create and set the value of an NVRAM variable
```a
BOOL SetFirmwareEnvironmentVariableA(
  LPCSTR lpName,
  LPCSTR lpGuid,
  PVOID  pValue,
  DWORD  nSize
);
```
2. [GetFirmwareEnvironmentVariable()](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getfirmwareenvironmentvariablea) - To fetch the value of an NVRAM variable
```a
DWORD GetFirmwareEnvironmentVariableA(
  LPCSTR lpName,
  LPCSTR lpGuid,
  PVOID  pBuffer,
  DWORD  nSize
);
```

So as can be seen from the MSDN docs GUID(Globally Unique Identifier) is basically a way to identify the specific variable in question. Each variable must have a name 

