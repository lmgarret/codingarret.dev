---
title: "Bazzite - Post-installation tweaks" # in any language you want
cover:
  image: "images/posts/bazzite-post-install/cover.jpg"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
description: >
    A collection of post-installation tweaks for Bazzite 
showToc: true
tocOpen: true
author: LM. Garret
UseHugoToc:
---


## Set the correct keyboard for first SDDM start and LUKS decryption prompt
One subtle but annoying problem with many Fedora Silverblue installations is that by default the login screen (SDDM) and the disk decryption prompt (LUKS) use the `en_US` default keyboard layout. This feels like an ever lasting Linux problem that will always need some manual tweaking to be fixed. Here is how we can address that:

First, look for the keymap you want to use with:

```command
localectl list-keymaps| grep CH
de_CH-latin1
fr_CH
fr_CH-latin1
mac-de_CH
mac-fr_CH-latin1
```

I'll be using `fr_CH` on this device, but of course any locale should work. 
Set your keymap with:
```command
sudo localectl set-keymap fr_CH
```

The changes should be visible in `/etc/vconsole.conf`
```command
cat /etc/vconsole.conf                                                 ✔
# Written by systemd-localed(8) or systemd-firstboot(1), read by systemd-localed
# and systemd-vconsole-setup(8). Use localectl(1) to update this file.
KEYMAP=fr_CH
FONT=eurlatgr
XKBLAYOUT=ch
XKBMODEL=pc105
XKBVARIANT=fr
XKBOPTIONS=terminate:ctrl_alt_bksp
```

and `/etc/X11/xorg.conf.d/00-keyboard.conf`, amongst probably other places

```command
cat /etc/X11/xorg.conf.d/00-keyboard.conf
# Written by systemd-localed(8), read by systemd-localed and Xorg. It's
# probably wise not to edit this file manually. Use localectl(1) to
# update this file.
Section "InputClass"
        Identifier "system-keyboard"
        MatchIsKeyboard "on"
        Option "XkbLayout" "ch"
        Option "XkbModel" "pc105"
        Option "XkbVariant" "fr"
        Option "XkbOptions" "terminate:ctrl_alt_bksp"
EndSection
```

This should allow having the correct keyboard layout for the login screen (SDDM) when booting.

Now to also set the right keyboard layout for the LUKS decryption prompt, run:
```command
sudo rpm-ostree initramfs-etc --track=/etc/vconsole.conf
```
and reboot. Your changes should now be applied!


## Change the SDDM login screen wallpaper
I dislike the default login screen wallpaper, so I wrote an [entire post here](/posts/bazzite/custom-sddm-theme/) on how to create your own SDDM custom theme in order to change the wallpaper.

{{< figure class="center" src="/images/posts/custom-sddm-theme/sddm-settings.png" alt="SDDM Settings" width="400px" caption="Our custom theme is here!">}}

## (Fedora 41) Fix Breeze cursor theme scale
On terminal, the cursor is comically huge. Fixed it thanks to [this blogpost](https://bbs.archlinux.org/viewtopic.php?pid=2199244#p2199244), adapted for Fedora.

In a toolbox, install the needed system dependencies
```command
sudo dnf install inkscape xcursorgen mesa-libGL mesa-libEGL
```

and then the following pip package
```command
pip install pyside6-essentials
```

Now clone the `breeze` repo (FYI it's 1.5GB):
```command
git clone https://github.com/kde/breeze
```

Edit `breeze/cursors/src/build.sh`, change `NOMINAL_SIZE=24` to `NOMINAL_SIZE=32`.

Build the cursor theme:
```command
cd breeze/cursors/Breeze && ../src/build.sh
```

TODO: rename theme
TODO: create archive and install cursor theme

## Fix DNS lookup timeout slowing down apps and updates

DNS may lookup host names that it can not find at usual DNS servers. Fix by editing /etc/hosts

```plaintext {title="/etc/hosts"}
# add the following at the end
127.0.0.1   <your_hostname> fedora bazzite toolbox toolbx 
```