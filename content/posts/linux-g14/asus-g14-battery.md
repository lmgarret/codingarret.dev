---
title: "Linux battery life on an Asus G14" # in any language you want
cover:
  image: "images/posts/asus-g14-battery/cover.jpg"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
description: >
    A short guide to improve the battery life on Asus Zephyrus G14 running Linux.

---

## Context
> TLDR; Battery life on the G14 has always been a struggle, on Windows or Linux, but we can improve it. I'm using [Aurora](https://getaurora.dev/) but this should apply to all modern Linux distros.


Hi! After letting my laptop live a long and peaceful Windows-life, I wanted to finally make the switch to Linux for this great machine. I'm already familiar with Linux, having used it for my studies, professionally, my HomeLab.. I used to go for the more familiar Ubuntu/Linux Mint/Debian distros, but this time I wanted to give Fedora a try. I have been reading a lot about atomic desktops, I'm quite fond of KDE on the Steam Deck, and I mainly use my laptop for development. So [Aurora](https://getaurora.dev/) looked like a good fit! (spoiler: it is).

I'm not going to detail the steps to install Aurora itself on the laptop, there are plenty of other resources covering this extensively. And this laptop is no different when it comes to dual-booting. What I wanted to cover in this post is a pain point many have encountered, initially on Windows but also once having installed Linux: battery life.

The G14 was marketed as a long battery life gaming laptop, which was a killer feature at the time! But there was a catch: the software quality was (is!) inferior to the hardware quality. After a few BIOS updates, the status quo seemed to be:
 - battery drain could go under ~-7w with no workload
 - suspend would cause a much higher battery drain when resuming. Only hibernating or rebooting would allow a low battery drain

Switching to Linux unfortunately comes with the same symptoms: the battery drain would remain high out of the box, also causing the fans to kick in permanently..

I'm going to share here two ways you can reduce this energy consumption to reach reasonnable levels (~-8w).

## Turning off the Nvidia GPU
The model I have comes with a RTX 2060 Max-Q. To be honest, I don't use it much anymore, thanks to getting Steam Deck. So personnally I chose to disable it altogether, but with a friendly tool to restore it.

> I have briefly used Bazzite on this laptop as well. Under Bazzite, I could properly switch to integrated graphics with just
```bash
supergfxctl -m Integrated
```
> and logging out.

We'll be using [envycontrol](https://github.com/bayasdev/envycontrol) for this task. This tool supports a lot Linux distributions, including Fedora Atomic Desktop! Follow [this link](https://github.com/bayasdev/envycontrol?tab=readme-ov-file#%EF%B8%8F-getting-envycontrol) to install it, then run

```bash
sudo envycontrol -s integrated
``` 
to use only the AMD integrated iGPU. In our case with the Aurora installation, it will add a layer to our image, effectively disabling the Nvidia dGPU. After a reboot, one can check that the mode is correctly set by querying:
```bash
$ sudo envycontrol -q
integrated
```

This was fairly simple, and you should already hear the silence coming out of the GPU fan! But we can further improve the laptop's battery life with this second tweak.

## Enabling the AMD PState EPP scaling driver
Scary name isn't it? We'll try to keep it simple