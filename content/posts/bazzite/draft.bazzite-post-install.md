# Bazzite post installation tweaks

## Set the correct keyboard for first SDDM start and LUKS decryption prompt
First, look for the keymap you want to use with:

```command
localectl list-keymaps| grep CH
de_CH-latin1
fr_CH
fr_CH-latin1
mac-de_CH
mac-fr_CH-latin1
```

Then set your keymap with:
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
