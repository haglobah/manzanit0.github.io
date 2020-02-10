---
layout: post
title: "Creating a bootable Linux USB stick"
author: Javier Garcia
category: Linux
tags: linux, ubuntu, bootable
---

Today I learnt how to create a bootable Linux USB stick from the terminal,
no fuss at all. In my case I tried it with Ubuntu 18.04 LTS.

```bash
df # Check mounted disks
sudo umount /dev/sda1 # Unmount your stick
sudo dd bs=4M if=/home/manzanit0/Downloads/ubuntu-18.04-desktop-amd64.iso of=/dev/sda1 status=progress oflag=sync # Write the image
```