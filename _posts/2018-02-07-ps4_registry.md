---
layout: page
title: PS4 Registry Editor
---

Recently I reverse-engineered PS4 registry format and made a simple tool to view and edit it. 

**Sony likes to encrypt and obfuscate everything.**
**Every time it is very fun to figure it out.**

PS4 registry is represented by a few different files and file formats:

 - /system_data/settings/system.nvs
 - /system_data/settings/system.dat
 - /system_data/settings/system.idx
 - /user/settings/system.eap
 - /system_data/settings/system.rec

The most important are system.dat and system.idx. The file format is simple: system.idx contains info about every entry with offset and system.dat contains entries data.

<div align="center">
    <img src ="/assets/ps4_reg_1.png"/>
</div>

Entries inside system.eap and system.rec are stored in obfuscated format.

<div align="center">
    <img src ="/assets/ps4_reg_2.png"/>
</div>

First of all, its XOR'ed with 8 bytes, there are a lot of Null bytes, so it's easy to figure out them without reversing. But thats not all.
RegID's are encrypted, entries and data are hashed. Besides that, RegID's for system.eap and system.rec are encrypted with different keys.

<div align="center">
    <img src ="/assets/ps4_reg_3.png"/>
</div>

Why to obfuscate? Dunno.

Folders with those files should be not easily accessible.

My suggestions:
 - To prevent fuzzing of file format
 - Implement new crypto and hash algorithms is the most fun thing to do at work
 - This data is very sensitive

It becomes even more fun when you find a backdoor that grants access to registry for non-system processes.
For it to work RegID's should be encrypted in another fashion.

<div align="center">
    <img src ="/assets/ps4_reg_4.png"/>
</div>

My tool allows to work with system.dat, system.idx, system.eap and system.rec.
System.nvs is not supported because its stored in kernel like view and therefore parsing will very depend on a system version. However, it contains the same entries as in system.rec.

It does not support rebuilding, so it is not possible to add new entries yet. 

As of 5.01 system.rec seems to contain additional layer of encryption, it's not implemented yet.

<div align="center">
    <img src ="/assets/ps4_reg_5.png"/>
</div>

[Source](https://github.com/oct0xor/ps4_registry_editor)

{% include twitter_plug.html %}