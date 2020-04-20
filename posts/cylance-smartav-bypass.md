# How I bypassed "next-generation" Cylance Smart AntiVirus in less than 15 minutes

**Reading Time:** 

## Prologue
Hello folks, In this blog-post, I am going to show you guys how I was able to bypass a "next-generation" Antivirus named Cylance Smart AV which supposedly uses neural networks for threat detection compared to traditional detection mechanisms.

Disclosure: This attack would specifically work on the Home edition of CylancePROTECT which is also known as Smart AV and not the Enterprise edition. We'll discuss this later on in this post but for now, fellow readers be forewarned.

Without any further ado, let's begin our analysis.

## Introduction
So this time I decided to try and perform a simulated attack scenario against a host protected by Cylance Smart AV. 

![Cylance SmartAV logo](../assets/images/cylance-smartav-logo.png "Cylance SmartAV logo")

For those of you who haven't heard about Cylance, do not be dismayed. Let me fill you in. Cylance(now acquired by BlackBerry Limited) belongs to one of the newer waves of security solutions that add a pinch of Machine Learning to existing detection algorithms. Basically, it employs ML models which are trained with millions of malicious samples to detect the next threat that comes around even if it's previously unseen. All of this is done in the hopes of removing too much dependency on signature-based detection models which have to be frequently updated to keep up with the recent threats and even then it can't possibly keep up its pace with zero-day threats and previously unseen or evolved malware samples.

I would recommend checking their page for more intel on their product which you can do [here](https://shop.cylance.com/us/).

These are company claims anyway. But is it effective? How well does it perform in a real-world assessment? Let's subject it to such a test to determine that answer, shall we?

I have tried to keep the scenario as real and practical as possible which initiates from the Delivery phase of the Cyber Kill Chain(if you don't know what that is [here](https://www.lockheedmartin.com/en-us/capabilities/cyber/cyber-kill-chain.html) is a link to read up on it).

This is not one of those impractical posts showing the results of a VirusTotal scan against a freshly-coded simple reverse shell to judge whether it'll pass as the payload for an actual red-teaming engagement and if so terming it as a bypass.

Before we begin I want to make it clear that I was neither employed by any company nor paid by an individual to perform these tests. All of this was done on personal interest and curiosity.
