# Introducing MIDNIGHTTRAIN - A Covert Stage-3 Persistence Framework utilizing UEFI variables

**Reading Time:** _16 minutes_

## TL;DR
This tool is [![ForTheBadge built-with-love](http://ForTheBadge.com/images/badges/built-with-love.svg)](https://GitHub.com/Naereen/) for red team operators, offensive security researchers and defenders alike ergo, being made open source and free(as in free beer).

It is available here: [https://github.com/slaeryan/MIDNIGHTTRAIN](https://github.com/slaeryan/MIDNIGHTTRAIN)

Fair warning: This has been made as a small weekend project and while the code has been tested in my lab extensively, bugs are to be expected. Furthermore, I'm not a professional coder so get ready to read crappy code! However, I have given it my best possible efforts and if you find bugs/improvements in the code don't feel shy to contact me!

## Introduction
One of my favourite pastimes is to read APT reports and look for the interesting TTPs(Tactics, Techniques and Procedures) used by apex adversaries(Read as: State-backed Threat Groups) in a later attempt to recreate it or at least a variant of it in my lab.

So last Friday was no different, except that this time I was going through the _CIA Vault7_ leaks specifically the _EDB_ branch documents when suddenly [this](https://wikileaks.org/ciav7p1/cms/page_26968084.html) document came to my attention which describes the theory behind NVRAM variables.
Immediately it piqued my interest and I started digging deeper.

Turns out, these variables are not only writable/readable from the user-mode but also it's an awesome place to hide your shit like egress implants, config data, stolen goodies and whatnot which I found out after watching an enlightening [DEFCON talk](https://youtu.be/q2KUufrjoRo), thanks to Topher Timzen and Michael Leibowitz.

In the talk, they do give a demo using C# but the attendees are encouraged to figure out their own way to weaponize this technique.

So it got me thinking of various ways to weaponize this and suddenly I remembered glossing over a report by [ESET](https://www.welivesecurity.com/2019/11/21/deprimon-default-print-monitor-malicious-downloader/) some time back describing an _alleged CIA_ implant(Ironically again!) named **DePriMon** which registered as the default print monitor to achieve persistence on the host(hence the name).

That was the birth of the **MIDNIGHTTRAIN** framework. Over the next two days, I spent time coding it and then a couple of days more for cleaning up and writing this post.

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
1. [GetFirmwareEnvironmentVariable()](https://docs.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-getfirmwareenvironmentvariablea) - To fetch the value of an NVRAM variable
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

Using these API functions require that you are a **local admin** and that you have a specific privilege available and enabled in the calling token namely - `SeSystemEnvironmentPrivilege/SE_SYSTEM_ENVIRONMENT_NAME`. This means that our persistence framework won't install without an **Elevated Context**.(Blue Teams take note!)

I wouldn't consider this a huge problem for attackers since persistence is typically meant to be a Post-Ex job and could be easily installed after privilege escalation on the host.

But the problem doesn't end there. The next big caveat is size. How much data can you store in a single NVRAM variable and how many such variables can be created reliably?

That shall solely dictate what can or can't be used as the payload.

To answer this question, I have done some testing in my lab and I have found that you can approximately create around **50 variables** and each with a capacity of **1000 characters** before Windows starts whining with a **1470** error code.

Alos, now is a good time to point out that it is possible to enumerate these variables from **Kernel-mode i.e. Ring 0** using frameworks such as [CHIPSEC](https://github.com/chipsec/chipsec) or using **physical access** to the machine with an **UEFI shell**(Again, Defenders take note!)

### Port Monitors
Once again, I will not bore you with endless theory. But it is important to know a few things. Port Monitors are User-mode DLLs that according to [MSDN](https://docs.microsoft.com/en-us/windows-hardware/drivers/print/port-monitors) "are responsible for providing a communications path between the user-mode print spooler and the kernel-mode port drivers that access I/O port hardware".

These DLLs are loaded by the **Print Spooler Service** or `spoolsv.exe` at startup and for that to happen primarily one of the two methods must be followed:

1. The fully-qualified pathname of the DLL must be written to `HKLM\SYSTEM\CurrentControlSet\Control\Print\Monitors`

![Port Monitor Registry Entry](../assets/images/port-mon-reg.png "Port Monitor Registry Entry")

This requires either a manual registry entry or via WinAPI and it allows loading of arbitrary DLLs.
1. The second method has a couple of more constraints.

- The DLL must reside in `System32`
- Arbitrary DLLs cannot be loaded via this technique(well, it can but without persistence), the DLL must be written in a [special way](https://docs.microsoft.com/en-us/windows-hardware/drivers/print/port-monitor-server-dll-functions) with some mandatory functions defined and must export a function named `InitializePrintMonitor2` which gets called immediately after the DLL is loaded

Finally, the Port Monitor might be registered via:

[AddMonitor()](https://docs.microsoft.com/en-us/windows/win32/printdocs/addmonitor?redirectedfrom=MSDN) - To  install a local port monitor
```a
BOOL AddMonitor(
  _In_ LPTSTR pName,
  _In_ DWORD  Level,
  _In_ LPBYTE pMonitors
);
```

What the function does under the hood is add the same registry entries and load the DLL within `spoolsv.exe` but without any direct intervention. Readers should probably take note that if the DLL is not created exactly according to MSDN specifications then while the DLL will be loaded for the current session but the appropriate registry entries will **not** be made and the DLL will **not load** after a reboot ergo, defeating it's very purpose.

And to uninstall a Port Monitor:

[DeleteMonitor()](https://docs.microsoft.com/en-us/windows/win32/printdocs/deletemonitor) - To remove a local port monitor
```a
BOOL DeleteMonitor(
  _In_ LPTSTR pName,
  _In_ LPTSTR pEnvironment,
  _In_ LPTSTR pMonitorName
);
```

I have chosen the second way for the framework.

### Execution Guardrails with DPAPI
Well, I'm a big fan of putting execution guardrails in my code primarily because of two reasons:

1. To prevent the accidental breaking of the rules of engagement. This will ensure that our malcode doesn't end being executed on any unintended host which are out of the scope
1. To hinder the efforts of blue teams trying to reverse engineer the implant on non-targeted assets and thwart analysis on automated malware sandboxes

Although, I think in this case the latter is more applicable than the former.

So what's DPAPI?

DPAPI(Data Protection API) is simply a set of functions provided by Microsoft intended to ensure confidentiality and integrity of locally stored credentials like Browser passwords, WiFi PSKs etc.

This is primarily achieved through the use of two functions:

1. [CryptProtectData](https://docs.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptprotectdata)
```a
DPAPI_IMP BOOL CryptProtectData(
  DATA_BLOB                 *pDataIn,
  LPCWSTR                   szDataDescr,
  DATA_BLOB                 *pOptionalEntropy,
  PVOID                     pvReserved,
  CRYPTPROTECT_PROMPTSTRUCT *pPromptStruct,
  DWORD                     dwFlags,
  DATA_BLOB                 *pDataOut
);
```
2. [CryptUnprotectData](https://docs.microsoft.com/en-us/windows/win32/api/dpapi/nf-dpapi-cryptunprotectdata)
```a
DPAPI_IMP BOOL CryptUnprotectData(
  DATA_BLOB                 *pDataIn,
  LPWSTR                    *ppszDataDescr,
  DATA_BLOB                 *pOptionalEntropy,
  PVOID                     pvReserved,
  CRYPTPROTECT_PROMPTSTRUCT *pPromptStruct,
  DWORD                     dwFlags,
  DATA_BLOB                 *pDataOut
);
```

Apart from the fact that these functions are quite straightforward to use, it provides another benefit.

If we can encrypt a data blob with DPAPI on the target host, that encrypted data cannot be decrypted anywhere else but on the same host machine. This means that if a payload is encrypted directly on a targeted asset, it shall make decryption and ergo execution non-trivial on a non-targeted asset like a sandbox or say a malware analyst's VM.

I got the inspiration for this from a malware named **InvisiMole**, technical analysis courtesy of [ESET](https://www.welivesecurity.com/wp-content/uploads/2020/06/ESET_InvisiMole.pdf).

You can read in-detail about DPAPI if you're interested [here](https://docs.microsoft.com/en-us/previous-versions/ms995355(v=msdn.10)?redirectedfrom=MSDN#windataprotection-dpapi_topic04).

### Thread Hijacking
Typical code injection uses thread injection using the documented `CreateRemoteThread()` or it's lesser known undocumented cousin `RtlCreateUserThread()` or an Nt* equivalent in Ntdll like `NtCreateThreadEx()`.

What happens in Thread Injection is that a thread is created in the remote process to run our malcode.

Though this remains one of the most popular, easy to implement and stable forms of code injection, this actually has some disadvantages from an OPSEC perspective. With tools, such as [Get-InjectedThread](https://gist.github.com/jaredcatkinson/23905d34537ce4b5b1818c3e6405c1d2) it is quite easy to detect an injected thread in a remote process by spotting missing `MEM_IMAGE` flags for the memory of the thread start address.

Anyway, this is something that [@xpn(Adam Chester)](https://blog.xpnsec.com/undersanding-and-evading-get-injectedthread/) will do a far better job of explaining than me.

The way Thread Hijacking overcomes the obstacle is by not injecting a thread in the first place but instead hijacking an existing thread of the remote process by first suspending it, then redirecting the `RIP` register to our malcode before resuming the thread again to launch our malcode this time.

This is why it is also fondly known as SiR(Suspend-Inject-Resume) injection. Pretty neat eh?

To accomplish this, we primarily need to perform the following steps(and API calls):

1. [VirtualAllocEx()]() - To allocate memory in the target process for our shellcode
2. [WriteProcessMemory()]() - To write the shellcode to the allocated memory in the target process
3. [SuspendThread()]() - To suspend a thread
4. [GetThreadContext()]() - To fetch the current state of the registers for our hijacked thread
5. [SetThreadContext()]() - To set the updated state of the registers for our hijacked thread specifically the RIP register now redirected to point to our shellcode
6. [ResumeThread()]() - To resume the hijacked thread



