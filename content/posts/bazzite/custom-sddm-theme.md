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

Fedora Silverblue images are great. However, there are sometimes small itches that you need to scratch, and having an immutable distro can get in your way. In this post, I'll cover a minor itch I wanted to scratch quickly after installing Bazzite: the login screen theme.

After booting, you are prompted with a screen to select the user you want to open a session for. This screen is rendered by [SDDM](https://github.com/sddm/sddm/). While it looks nice and modern, it may not be exactly to your taste. In my case, I didn't like the default wallpaper, but this really comes down to personal preference.

{{< figure class="center" src="/images/posts/custom-sddm-theme/f40-01-day.png" alt="Bazzite default SDDM wallpaper" width="400px" caption="Yeah, not my favorite wallpaper for a gaming rig...">}}

I explored various ways to customize the SDDM theme and decided to create a new theme based on the default one out of simplicity. This post will guide you through the process of building a customized SDDM theme.

So, let's get started!

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

**Make sure to rename the theme** to something else to avoid conflicts, by editing the `Name` field in `metadata.desktop`. e.g. to `Breeze Fedora (Custom)`.

{{< notice note >}}
2024.10.20: Since a recent update, if you are using the "Breeze" theme or one of its variants, you may encounter the `Theme has an unknown property "fontSize"` error on your SDDM screen. This is due to changes in the API that the Breeze theme has not yet been updated to accommodate.

You can follow this tutorial to fork the theme and edit the `Main.qml` file. Replace every occurrence of:
- `fontSize` with `font.pointSize` (except on line 327)
- `iconSource` with `icon.name`.

{{< /notice >}}

## Installing the theme
We will explore two options here:
 - the easy one: packaging an archive that we can manually install in the KDE SDDM settings
 - the involved one: building a RPM out of our theme to layer it in our system

### Option 1: building a theme file + bind mount
This is the fairly easy option. We first create a theme archive. Then we need to workaround a [`SDDM` limitation](https://pagure.io/fedora-kde/SIG/issue/282): by default, the `SDDM` theme installer installs themes to the read-only `/usr/share/sddm` directory. Attempting to install the theme directly would throw us `Could not decompress the archive`. The [chosen workaround](https://discussion.fedoraproject.org/t/another-way-to-customize-sddm-under-kinoite/37773) is to mount bind a writable directory in-place of the default theme directory (`/usr/share/sddm`).

Here are the steps to follow:

1. In your working directory, package the theme archive with:
```command
tar -czvf breeze-custom.tar.gz 02-breeze-fedora-custom
```

2. Copy the existing theme dir to a new (read-writable) one:
```command
sudo cp -r /usr/share/sddm /var
```

3. Edit `/etc/fstab` to mount bind your new directory in-place of the default themes one. At the end of the file, add 
```
/var/sddm /usr/share/sddm none rbind 0 0 
```

4. Remount your partitions with
```command
sudo mount -a
```

5. Install the theme file. Head over to the KDE Settings app, search for `SDDM`. On the SDDM settings screen, click on `Install from File...`, and select the created archive.

{{< figure class="center" src="/images/posts/custom-sddm-theme/sddm-settings-install.png" alt="SDDM Settings" caption="'Install from File' in the top right corner">}}

Enter your sudo password when prompted, then hit `Apply`, re-enter your password, and you're done! You can now log out of your session, and you should see your customized SDDM theme in action.

### Option 2: as a RPM file
Okay this one might be a bit too involved and unnecessary, I'll share it nonetheless since some may be interested in layering the theme into their image.

What we are going to do here is build a different theme archive, then create a RPM file out of it using `sddm2rpm`. Then we'll install it!

#### Installing SDDM2RPM
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

#### Preparing the archive
Go to your working directory
```command
cd $HOME/custom-sddm-theme
```

and prepare the archive. Note that the command is slightly different from the Option 1:
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

#### Installing the package
In a new shell (outside the toolbox), install the package with
```command
sudo rpm-ostree install ./sddm-breeze-custom.rpm
```

At the end of the installation it should prompt you to reboot, which you have to do to apply the changes.

After rebooting, you can open the `Login Screen (SDDM)` app to see a list of installed themes. Your theme should appear here!


{{< figure class="center" src="/images/posts/custom-sddm-theme/sddm-settings.png" alt="SDDM Settings" caption="Our custom theme is here!">}}

Since we layered our theme, it should persist across system updates.
