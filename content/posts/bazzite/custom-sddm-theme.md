---
title: "Customizing a SDDM theme" # in any language you want
cover:
  image: "images/posts/custom-sddm-theme/cover.jpg"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
description: >
    A guide to customize an existing SDDM (login screen) theme
showToc: true
tocOpen: true
author: LM. Garret
UseHugoToc:
---

Fedora Silverblue images are great. But there will sometimes small itches that you need to scratch, and having an immutable distro can be in your way. In this post, I'll cover a minor itch I wanted to scratch quickly after installing Bazzite: the login screen theme.


After booting, you are prompted with a screen to select the user you want to open a session for. This screen is rendered by [SDDM](https://github.com/sddm/sddm/). While it looks nice and modern, it may not be exactly fitting your tastes. In my case, I didn't like the default wallpaper, but this really comes down to personal taste.


{{< figure class="center" src="/images/posts/custom-sddm-theme/f40-01-day.png" alt="Bazzite default SDDM wallpaper" width="400px" caption="Yeah, not my favorite wallpaper for a gaming rig...">}}

I looked for ways to customize the SDDM theme. There are multiple options here, but I went for the "create a new theme based on the default one" out of curiosity, and this is what I'll be covering in this post.

So, let's build a customized SDDM theme!


## Installing SDDM2RPM
To edit the default theme, we will be using sddm2rpm. Since we're in an immutable distro, we will first install its dependencies in a new toolbox container which we will call `sddm2rpm`:
```command
toolbox create sddm2rpm
Created container: sddm2rpm
Enter with: toolbox enter sddm2rpm
```

Then we enter the toolbox
```command
toolbox enter sddm2rpm
$
```

And install cargo
```command
curl https://sh.rustup.rs -sSf | sh
info: downloading installer
...
```

When asked
```console
1) Proceed with standard installation (default - just press enter)
2) Customize installation
3) Cancel installation
```
you can go with the standard installation and press enter.

Once installed, you either need to add the installation directory it prints you to your `PATH`, or simply restart your shell.

Now we can install gcc:
```command
sudo dnf install gcc -y
```

And finally, clone the project's repo in our container:

```command
git clone https://github.com/Lunarequest/sddm2rpm.git
```

Now to build and install sdd2rpm:
```command
cd sdd2rpm
```

```command
cargo install --path .
```

`sdd2rpm` should now be available in your path - let's build the theme!

## Creating the SDDM theme
Prepare a temporary directory to create the theme:
```command
mkdir -p $HOME/custom-sddm-theme
```
then
```command
cd $HOME/custom-sddm-theme
```

Then copy over the files from the theme you want to modify. In my case, I want to keep the default breeze fedora theme, while changing the background image. So I'll first copy the theme folder that I want, giving it a new name
```command
cp -r /usr/share/sddm/themes/01-breeze-fedora/ 02-breeze-fedora-custom
```

I can now change the background image in `theme.conf`, replacing `background` value with the path to the new image to use.

**Make sure to rename the theme** to something else to avoid conflicts, by editing the `Name` field in `metadata.desktop`. e.g. `Breeze Fedora (Custom)`.

Once happy with the changes, prepare the archive:
```command
tar -czvf sddm-breeze-custom.tar.gz -C 02-breeze-fedora-custom .
./
./default-logo.svg
./metadata.desktop
./preview.png
./Background.qml
./Login.qml
./Messages.sh
./KeyboardButton.qml
./SessionButton.qml
./Main.qml
./faces/
./faces/.face.icon
./theme.conf
```

Everything is now ready to create our theme package. Create the RPM file with
```command
sddm2rpm sddm-breeze-custom.tar.gz --pkg-version=1.0
```

Note that you **must** be in the tar archive directory when running this command. Passing `$HOME/custom-sddm-theme/sddm-breeze-custom.tar.gz` **will fail** at install time as the generated package will retain the path. This is an unfortunate bug that can be easily worked around by ensuring you are in the same directory as your archive.

## Installing the theme
In a new shell (outside the toolbox), install the package with
```command
sudo rpm-ostree install ./sddm-breeze-custom.rpm
```

At the end of the installation it should prompt you to reboot, which you have to do to apply the changes.

After rebooting, you can open the `Login Screen (SDDM)` app to see a list of installed themes. Your theme should appear here!


{{< figure class="center" src="/images/posts/custom-sddm-theme/sddm-settings.png" alt="SDDM Settings" caption="Our custom theme is here!">}}

Since we layered our theme, it should persist across system updates. But keep in mind that SDDM updates may break it, in which case you should address any new modifications in your theme.