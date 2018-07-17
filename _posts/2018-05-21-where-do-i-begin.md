---
layout: post
published: true
title: Where do I begin?
date: '2018-05-26'
subtitle: Building a lab (that's NOT prod)
---
Ever since [MMS](https://mmsmoa.com/), I knew I needed to start contributing.  I kept asking "where do I begin?"  And it hit me: that's what EVERY aspiring admin asks.  Where do I begin?

This will be a series of posts where we create a simple lab and mature it with the fundamentals of what SCCM can do.

### The Overview
1. [Stand up an SCCM lab in Hyper-V](https://doug.seiler.us/2018-05-30-set-up-the-sccm-lab/).  This means a domain controller running DHCP and an SCCM server running current branch.
2. [Set up OSD (operating system deployment) over PXE](https://doug.seiler.us/2018-06-05-you-down-with-osd/).  Now we have an environment.
3. [Install WSUS and start patching with SCCM](https://doug.seiler.us/2018-07-10-SCCM-WSUS-Patching/).  If you have an environment, we'll need to patch it.
4. Application deployment.  On the off chance your environment needs software NOT in the Micorsoft Store.
5. Security.  Group Policy, Secure Boot, Bitlocker, Credential Guard, Device Guard, Exploit Guard, Applocker, Defender, and maybe ATP.

### The Guarantee
Why take my word for it?  Don't.  No need to put any faith in me, I'm stealing all of this.  The community is full of tools, ideas, and great documentation.

### The Requirements
1. Hardware.  A desktop with a half decent CPU, 16GB of RAM, and at least 500GB of disk.
2. ISOs. Windows 10, and Windows 7 SP1 if you're a masochist (or testing upgrades).  Preferably Enterprise editions, but we don't need any licenses.
3. Miscellaneous downloads.  Totally free, we'll get them as we go.
