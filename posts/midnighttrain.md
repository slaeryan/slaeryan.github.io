# Introducing MIDNIGHTTRAIN - A Covert Stage-3 Persistence Framework utilizing UEFI variables

**Reading Time:** _16 minutes_

## TL;DR
This tool is [![ForTheBadge built-with-love](http://ForTheBadge.com/images/badges/built-with-love.svg)](https://GitHub.com/Naereen/) for red team operators, offensive security researchers and defenders alike ergo, being made open source and free(as in free beer).

It is available here: [https://github.com/slaeryan/MIDNIGHTTRAIN](https://github.com/slaeryan/MIDNIGHTTRAIN)

Fair warning: This has been made as a small weekend project and while the code has been tested in my lab extensively, bugs are to be expected. Furthermore, I'm not a professional coder so get ready to read crappy code! However, I have given it my best possible efforts and if you find bugs/improvements in the code don't feel shy to contact me!

## Introduction
One of my favourite hobbies is to read APT reports and look for the TTPs(Tactics, Techniques and Procedures) used by apex adversaries(Read as: State-backed Threat Groups) and later attempt to recreate it or a variant of it myself in my lab.

So last Friday was no different, except that this time I was actually going through the _CIA Vault7_ leaks specifically the _EDB_ branch documents when suddenly [this](https://wikileaks.org/ciav7p1/cms/page_26968084.html) document came to my attention which describes the theory behind NVRAM variables.
Immediately it piqued my interest and I started digging deeper.

Turns out, these variables are not only writable/readable from the user-mode but also it's an awesome place to hide your shit like egress implants, config data, stolen goodies and whatnot which I found out after watching an enlightening [DEFCON talk](https://youtu.be/q2KUufrjoRo), thanks to Topher Timzen and Michael Leibowitz.

In the talk, they do give a demo using C# but the attendees are encouraged to figure out their own way to weaponize this technique.

So it got me thinking of various ways to weaponize this and suddenly I remembered glossing over a report by [ESET](https://www.welivesecurity.com/2019/11/21/deprimon-default-print-monitor-malicious-downloader/) some time back describing an _alleged CIA_ implant(Ironically again!) named **DePriMon** which registered as the default print monitor to achieve persistence on the host(hence the name).

That was the birth of the **MIDNIGHTTRAIN** framework. Over the next two days, I spent time coding it and then a couple of days more for writing this post.

## Of NVRAM variables, Print Monitors, Execution Guardrails with DPAPI, Thread Hijacking etc. oh my my!
For the uninitiated readers, don't be scared of these buzzwords for I can guarantee you that this is absolutely nothing to be scared of and I shall attempt to explain each of these individual components(and the motivation behind it) one by one.

But first, let's go through these basic concepts. 

Initiated readers, feel free to skip this part and move on directly to the framework architecture.

### NVRAM Variables
I am not going to bore you with the theory, for that is not my goal. Just know that all modern UEFI machines use these variables to store important boot-time data and various other vendor-specific data in the flash memory. Needless to say, this data will survive a full reinstallation of the Operating System and to quote the CIA, "are invisible to a forensic image of the hard drive".

Sound like a stealthy place to hide your life's secrets yet?

What's more? As easy as it is to write data into firmware variables from User-mode i.e. Ring 3, it is incredibly difficult(if not downright impossible) for the defenders to enumerate the data from the same.

How so you ask? Well, you'll see in a bit.

Now, conveniently for us attackers, Microsoft provides us with fully-documented API access to the magical land of firmware variables using:

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

And if you're wondering what a _Guid_ is, A GUID(Globally Unique Identifier) along with the variable name is just a way to identify the specific variable in question. Therefore, each variable must have a unique name and GUID.

Now does it make sense why it's almost impossible to enumerate from Ring 3? Because enumeration would require the exact name and GUID of the variables. Hell to even verify the existence of a variable, you'd need its specific name and GUID.

Okay, this sounds too good to be true so what's the caveat? Can you call these API's even from a non-elevated context?

Good question, the answer is no! 

Using these API functions require that you are a **local admin** and that you have a specific privilege available and enabled in the calling token namely - **SeSystemEnvironmentPrivilege/SE_SYSTEM_ENVIRONMENT_NAME**. This means that our persistence framework won't install without an **Elevated Context**.(Blue Teams take note!)

I wouldn't consider this a huge problem for attackers since persistence is typically meant to be a Post-Ex job and could be easily installed after privilege escalation on the host.

But the problem doesn't end there. The next big caveat is size. How much data can you store in a single NVRAM variable and how many such variables can be created reliably?

That shall solely dictate what can or can't be used as the payload.

To answer this question, I have done some testing in my lab and I have found that you can approximately create around **50 variables** and each with a capacity of **1000 characters** before Windows starts whining with a **1470** error code.

Alos, now is a good time to point out that it is possible to enumerate these variables from **Kernel-mode i.e. Ring 0** using frameworks such as [CHIPSEC](https://github.com/chipsec/chipsec) or using **physical access** to the machine with an **UEFI shell**(Again, Defenders take note!)

### Port Monitors

