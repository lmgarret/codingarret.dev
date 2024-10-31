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
Many Fedora Silverblue images use the `en_US` keyboard layout by default for the disk decryption prompt (LUKS) and the login screen (SDDM). This may not match your keyboard, so we'll change it to another layout.

1. Find your desired keymap:
```command
localectl list-keymaps| grep CH
de_CH-latin1
fr_CH
fr_CH-latin1
mac-de_CH
mac-fr_CH-latin1
```

2. Set your keymap (replace `fr_CH` with your desired keymap):
```command
sudo localectl set-keymap fr_CH
```

3. Verify the changes in `/etc/vconsole.conf`
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
 and `/etc/X11/xorg.conf.d/00-keyboard.conf`.
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

This should do it for SDDM.

4. Now to set the keyboard layout for the LUKS decryption prompt, run:
```command
sudo rpm-ostree initramfs-etc --track=/etc/vconsole.conf
```
5. Reboot for the changes to take effect.


## Change the SDDM login screen wallpaper
I dislike the default login screen wallpaper, so I wrote an [entire post here](/posts/bazzite/custom-sddm-theme/) on how to create your own SDDM custom theme in order to change the wallpaper.

{{< figure class="center" src="/images/posts/custom-sddm-theme/sddm-settings.png" alt="SDDM Settings" width="600px" caption="Our custom theme is here!">}}

## Fix Breeze cursor theme scale (Fedora 41)
{{< figure class="floatright" src="/images/posts/bazzite-post-install/big-cursor.png" alt="KDE Cursor Settings" width="300px" caption="Big and pixelated cursor">}}
There's a GTK bug that's causing the KDE cursor for some themes (Breeze..) to become huge and pixelated. This is notably visible in Bazzite's terminal app, as shown in the right picture:

A [MR](https://gitlab.gnome.org/GNOME/gtk/-/merge_requests/7760) fixing this bug has been merged, but it is planned for release in March 2025! 

For those like me who can't wait, here's a workaround I found thanks to [this blogpost](https://bbs.archlinux.org/viewtopic.php?pid=2199244#p2199244). The solution isn't perfect but is more acceptable than the status quo.

What we will do is simply rebuild the `Breeze` cursor theme, fixing its nominal size to fit the real cursor images size.

1. In a toolbox, install the required dependencies:
```command
sudo dnf install inkscape xcursorgen mesa-libGL mesa-libEGL
```

and then the following pip package
```command
pip install pyside6-essentials
```

2. Clone the `breeze` repo:
```command
git clone https://github.com/kde/breeze
```

3. Edit `breeze/cursors/src/build.sh`, change `NOMINAL_SIZE=24` to `NOMINAL_SIZE=32`.

4. Build the cursor theme:
```command
cd breeze/cursors/Breeze && ../src/build.sh
...
Generating cursor theme... DONE
Generating shortcuts... DONE
Copying Theme Index... DONE
COMPLETE!
```

5. Rename the created theme to avoid overwriting any existing theme:
```command
sed -i 's/^Name=.*/Name=BreezeCustom/' Breeze/index.theme
```

6. Package the theme:
```command
tar -czvf BreezeCustom.tar.gz Breeze
```

7. Finally, install the theme by going to KDE's cursor settings, `Install from File...` and selecting the created `BreezeCustom.tar.gz` file. Then select it and hit apply.
{{< figure class="center" src="/images/posts/bazzite-post-install/cursor-settings-screen.png" alt="KDE Cursor Settings" width="600px" caption="Installing the cursor theme from file">}}

And that's it! Your cursor should now have a better scaling on GTK windows. 

> It's possible that your cursor is now a bit smaller in these GTK apps. This may be related to fractional scaling not being applied on the cursor. I haven't investigating further since this now looks acceptable to me until we get the GTK patch

## Fix DNS lookup timeout slowing down apps and updates
An annoying issue I ran into was some commands and apps being excruciating slow:
 - Firefox would take minutes to launch
 - `dnf install` in toolbox containers would take minutes before outputting anything

After some investigation, I found out that this was related to DNS queries for localhost like names timing out. This could be due changing the default DNS to a private one (NextDNS in my case).

To address this, the simplest solution is to add these localhost names in `/etc/hosts` for instant resolution:

```plaintext {title="/etc/hosts"}
# add the following at the end
127.0.0.1   <your_hostname> fedora bazzite toolbox toolbx 
```

`<your_hostname>` can be found with `cat /etc/hostname`. `fedora` and `bazzite` are quite obvious. For toolbox container, I found out that the `/etc/hostname` in the containers would always be `toolbox`, which changed to `toolbx` with some release. There might be a similar solution for distrobox, but I haven't used it yet.