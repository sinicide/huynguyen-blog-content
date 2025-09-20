---
layout: blog
title: Setting up a Fresh Worktop
date: 2025-09-19T18:47:45-05:00
lastMod: 2025-09-19T20:50:24-05:00
categories:
  - personal
tags:
  - macOS
  - laptop
  - dotfiles
  - raycast
  - sublimetext
  - obsidian
  - alacritty
  - neovim
  - tmux
  - zoxide
  - fzf
description: When that new work laptop hits and you got to scramble and make sure you can get back up and running with minimal setup time.
disableComments: true
draft: false
---

Unlike most people who seem to work in Tech and have Tech blogs and/or a presence online, I work specifically in a Technical Support Role, so I don't need a lot of fancy things to write code or do my job specifically. But I do have specific workflows and apps that I use on a frequent basis to help me communicate to customers, replicate issues, review configuration and logs as well as knowledge management. Over the years the stuff I've added and configured has sort of gotten away from me so I've been delaying a work machine replacement for a long time now. But enough is enough and it's time to get things in order so that I can get going from 0.

## Essential Apps

- [Raycast](https://www.raycast.com/)
- [SublimeText](https://www.sublimetext.com/)
- [Obsidian](https://obsidian.md/)
- [Alacritty](https://alacritty.org/)

It's been a few years since I had to sort of start fresh, so I wanted to see if there's new apps to swap out for and turns out there was. At work I use a Mac and at home I use Linux, so I've tried to mirror as much of my essential apps as I can, but there are just a few apps that are specific to Mac that I need.

I was using Alfred as my App Launcher, Spectacle pretty heavily for window movement and using the default MacOS Workspaces for virtual desktops as I primarily only worked off the single screen on the laptop. I know, I know, that's probably weird as most people are gonna have at least a 2 monitor setup for work. It helps me to keep things compartmentalize and makes it easier for me to work from the desk, couch or coffee shop if I want to move around.

While I'm still using Workspaces as my main virtual desktop, I've moved away from Spectacle and started using Raycast. Raycast is like a Applauncher on crack, it also has Window Management and an awesome plug-able extensions store. There's also a text expander feature that I'm using for stuff that I type over and over everyday, like greetings, signatures, etc. Raycast is now replacing 2 apps for me and opening a door to new possibilities, I can't wait to check out what else is in the extensions store.

I've been using SublimeText now for reviewing configuration files, small log files and typing scratch notes as I investigate application issues. My favorite feature is the little mini map as I'm a visual person and being able to see the shape of a log file really helps to take notice of things that you can easily miss if you're only grepping for ERRORs. It also has a pretty sweet plugin system as well, which was extremely useful before I swapped to Obsidian for Knowledge Management.

Of course I've been using Obsidian for quite a while now as it's been the ultimate tool for Knowledge Management, Meeting Notes, Investigation notes, etc. Using Templater has really opened up automating a lot of my workflows here and I can't imagine going back to simple text editors.

In terms of the Terminal Emulator, I'm still using a Single Alacritty Window with Tmux for multi-terminal spaces and sessions. Sometimes I have to try and replicate a customer issue or bug and Tmux makes it easy to move around.

## Dotfiles

For the most part my dotfiles are nearly identical to my personal [dotfiles](https://github.com/sinicide/.dotfiles) except for a private sub repo that I'm using to include work specific configurations that differ from my personal ones. There's also a mac specific initialization script for homebrew apps that I need installed.

