---
layout: page
title: PS4 Registry Editor
---

I recently reverse engineered the PS4 registry format and created a simple tool to view and edit it.

**Sony loves to encrypt and obfuscate everything.**
**It's always very fun to solve this.**

The PS4 registry is represented by several different files and file formats:

 - /system_data/settings/system.nvs
 - /system_data/settings/system.dat
 - /system_data/settings/system.idx
 - /user/settings/system.eap
 - /system_data/settings/system.rec

The most important are system.dat and system.idx. The file format is simple: system.idx contains information about each entry with an offset, and system.dat contains the entry data.

<div align="center">
    <img src ="/assets/ps4_reg_1.png"/>
</div>

Entries inside system.eap and system.rec are stored in an obfuscated format.

<div align="center">
    <img src ="/assets/ps4_reg_2.png"/>
</div>

First, it is XORed with 8 bytes. There are a lot of zero bytes, so the XOR key is easy to extract without reversing. But that's not all.
RegID's are encrypted, entries and data are hashed. In addition, RegID's for system.eap and system.rec are encrypted with different keys.

<div align="center">
    <img src ="/assets/ps4_reg_3.png"/>
</div>

Why to obfuscate? I don't know. Folders containing these files should not be easily accessible.

My guesses:
 - To prevent fuzzing of the file format
 - Implementing new encryption and hashing algorithms is the most fun thing to do at work
 - This data is very sensitive

It gets even more fun when you find a backdoor that provides registry access to non-system processes.
For this to work, RegID's must be encrypted in a different way.

<div align="center">
    <img src ="/assets/ps4_reg_4.png"/>
</div>

My tool allows you to work with system.dat, system.idx, system.eap and system.rec files.
System.nv is not supported because it is stored in a kernel-like representation and therefore its parsing will be highly dependent on the system version. However, it contains the same entries as system.rec.

It does not support rebuilding, so adding new entries is not yet possible.

As of version 5.01, system.rec appears to contain an additional layer of encryption, but it has not been implemented yet.

<div align="center">
    <img src ="/assets/ps4_reg_5.png"/>
</div>

[Source](https://github.com/oct0xor/ps4_registry_editor)

{% include twitter_plug.html %}