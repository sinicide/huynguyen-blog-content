---
layout: blog
title: Moving Apartments and Moving the Homelab
date: 2024-09-15T20:44:14-0500
lastMod: 2024-09-15T20:44:14-0500
categories: self-hosted
tags:
  - self-hosted
  - truenas
  - desktop
description: The weird quirks of your servers when they get powered down for a move...
disableComments: true
draft: false
---

Sometimes you just encounter some of the weirdest quirks with your tech and homelab and it only becomes apparent when you power off, dismantle, and move the whole shebang. I've recently moved and had to dismantle, box up, and relocate everything. Now let me preface this by saying while I would like to physically clean my servers on a more frequent bases, it's generally difficult with the limiting space that I often find myself in with apartments. So most maintenance tends to happen during apartment moves or when something goes haywire. Most of my gear has not changed for several years and I often don't really buy new parts outside of my desktop and hard drives.

So with this move, I ended up encountering 2 major issues and I figured documenting them might be good in case someone else comes across similar issues and just needs some ideas to get the brain motors turning.

## Problem #1

Let's start with my main PC. I built this over 2 years ago, so here are the current specs.

- Gigabyte X570 Aorus Ultra Motherboard
- AMD Ryzen 9 5900X
- 64GB of RAM
- A bunch of SSDs
- 1x Nvidia GeForce RTX 3080 Ti powering 4 monitors
- 1x Nvidia Quadro M4000 for passthrough on VMs connected to a Wacom Cintiq 22HD
- OS: Archlinux

Maybe some day I'll go over my main setup in further detail, but for now. Let's go over the issue.
When powering on, the fans come on, but the BIOS doesn't POST and holding down the power button doesn't actually turn off the PC. I have to manually switch off the Power Supply. So already something weird is going on.

No clue what could have happened during the move, but my guess is that either something got dislodged or a build up of static charge or something has caused an issue. So let's go over some of the basic things I've tried.

- Re-seating the RAM modules, moving the 2 sticks around, starting with 1 stick only. No changes.
- Re-seating the 2 Graphics cards, swapping them around, leaving in only one and removing them both. Still no changes.
- I took out the CMOS battery, waited 30 seconds to a minute and re-added it back. Again still no changes.

At this point I was starting to think that maybe the motherboard had gone bad and that I would need a whole new board replacement. But out of a final act of desperation I decided to re-seat the CPU itself. Amazingly this worked and my BIOS posted. I still don't understand why re-seating the CPU did the trick, or perhaps by re-seating the CPU I had allowed some built up static to discharge or maybe the CMOS battery was finally able to dissipate?
Either way, I'm glad this was a no-cost fix.

## Problem #2

My NAS was not detecting any of the harddrives outside of the boot drives. For my setup, I'm running a 20 Hot-swap bay Norco Chassis, that I built well over a decade ago at this point. The only thing recent in this machine is the Hard Drives for the ZFS pools. Now normally I'd be really concern that the hard drives died, but not this time since I manually transported the hard drives myself, only the server was moved via hired movers. So I was 99.9% confident that the hard drives are not the issue here.
This means that the possible points of failure can be any device/component between the hard drive to the PCI-E slot. Since the boot drives are directly connect to the Sata ports on the motherboard. Let's go over how a storage hard drive is connected.

Hard Drive -> Backplane -> Sata to SFF Cable -> SAS Expander Card -> SFF to SFF Cable to HBA Card

So I worked backwards, simply unplugging and replugging in cables and re-seating the HBA and Expander cards. It may seem dumb but sometimes things get loose in a move and just reseating/replugging them back in can solve the issue.

This was not the case for me. So I then tried move the PCIE slots that the HBA and SAS cards were connected in around. Still no dice. So next step was to swap more cables around, still nothing really.

Finally by shear coincidence, I started to really pay attention to the hot swap caddies during boot up as they do contain indicator lights for power and activity. I realized at somepoint that I was not seeing any green activity light during the startup phase when TrueNAS is trying to import the zfs pool. Since the power light shown, I started to suspect that maybe it was a sata/SFF cable issue. I decided to skip the SAS expander entirely and plug Sata/SFF cable directly into the HBA card instead. Plugged in a spare hard drive and I found out it was an issue with the SAS expander not being recognized. I made sure I tested all combination of SFF slots on the SAS expander with the HBA card, but none of it connected.

Luckily the SAS expander was cheap and I was able to buy a used replacement from ebay, which is great because when I first put together this NAS, I also bought this used. So anytime I can used gear for cheap it's a win in my book.

- Intel RES2SV240 24 port 6Gbps Sata SAS controller for around 30 USD

The great thing is that this is just a drop in replacement, once I replaced the card, TureNAS was able to import the zfs pool and I was back in business.

For gear that I bought used 10+ years ago it still works like a champ!
