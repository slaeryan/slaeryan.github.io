# How I bypassed "next-generation" Cylance Smart AntiVirus in less than 15 minutes

**Reading Time:** __

## Prologue
Hello folks, In this blog-post, I am going to show you guys how I was able to bypass a "next-generation" Antivirus named Cylance Smart AV which supposedly uses neural networks for threat detection compared to traditional detection mechanisms.

Disclosure: This attack would specifically work on the Home edition of CylancePROTECT which is also known as Smart AV and not the Enterprise edition. We'll discuss this later on in this post but for now, fellow readers be forewarned.

Without any further ado, let's begin our analysis.

## Introduction
So this time I decided to try and run a simulated attack scenario against a host protected by Cylance Smart AV in an attempt to breach it.

![Cylance SmartAV logo](../assets/images/cylance-smartav-logo.png "Cylance SmartAV logo")

For those of you who haven't heard about Cylance, do not be dismayed. Let me fill you in. Cylance(now acquired by BlackBerry Limited) belongs to one of the newer waves of security solutions that add a pinch of Machine Learning to existing detection algorithms. Basically, it employs ML models which are trained with millions of malicious samples to detect the next threat that comes around even if it's previously unseen. All of this is done in the hopes of removing too much dependency on signature-based detection models which have to be frequently updated to keep up with the recent threats and even then it can't possibly keep up its pace with zero-day threats and previously unseen or evolved malware samples.

I would recommend checking their page for more intel on their product which you can do [here](https://shop.cylance.com/us/).

These are company claims anyway. But is it effective? How well does it perform in a real-world assessment? Let's subject it to such a test to determine that answer, shall we?

I did run the scenario and guess what? It was way easier than I thought and it didn't take me more than 15 minutes to prepare a lure mail with a payload stager that would download and execute our final stage payload to give us the shell.

Yes, I have tried to keep the scenario as real and practical as possible which initiates from the Delivery phase of the Cyber Kill Chain(if you don't know what that is [here](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) is a link to read up on it).

This is not one of those impractical posts showing the results of a VirusTotal scan against a scratch-coded simple reverse shell to judge whether it'll pass as the payload for an actual red-teaming engagement and if so terming it as a bypass.

Before we begin I want to make it clear that I was neither employed by any company nor paid by an individual to perform these tests. All of this was done on personal interest and curiosity.

## Preparing the test environment
Of course we would need to set up a test environment to run the scenario. Here are the things you'd need:

1. A latest version of Windows 10 build with all security hotfixes installed - This is going to be our target machine.
1. A Linux box loaded with Metasploit framework - The attacker machine.
1. A copy of Cylance Smart AV with all features enabled - You may get a "trial" version for a month(5 USD).

I'd strongly recommend to set up a Windows VM using VirtualBox/VMWare instead of using your production machine. Since there are plenty of resources available online on how to do so and considering the fact that there's not much to explain here pertaining the main subject, I will leave this step as an exercise for the readers.

Keep in mind, for simplicity's sake we are not using a VPS as the target machine. Therefore, it is your responsibility to set up the environment in such a way that both the machines are on the same local network.