1. Install SDDM2RPM
```command
git clone https://github.com/Lunarequest/sddm2rpm.git
```

2. Create toolbox
```command
toolbox create sddm2rpm
Created container: sddm2rpm
Enter with: toolbox enter sddm2rpm
```

3. Enter toolbox
```command
toolbox enter sddm2rpm
$
```

4. Install cargo
```command
curl https://sh.rustup.rs -sSf | sh
info: downloading installer
...
```

When asked
```command
1) Proceed with standard installation (default - just press enter)
2) Customize installation
3) Cancel installation
```
you can go with the standard installation and press enter.

Once installed, you either need to add the installation directory it prints you to your `PATH`, or simply restart your shell.

5. Install gcc
```command
sudo dnf install gcc -y
```

6. Now build and install sdd2rpm with
```command
cd sdd2rpm
```

```command
cargo install --path .
```

7. Prepare the sddm theme
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

8. Create the RPM
Everything is now ready to create our theme package. Create the RPM file with
```command
sddm2rpm sddm-breeze-custom.tar.gz --pkg-version=1.0
```

Note that you **must** be in the tar archive directory when running this command. Passing `$HOME/custom-sddm-theme/sddm-breeze-custom.tar.gz` **will fail** at install time as the generated pacakge will retain the path. This is an unfortunate bug that can be easily worked around by ensuring you are in the same directory as your archive.

9. Install the package
In your second shell (outside the toolbox), install the package with
```command
sudo rpm-ostree install ./sddm-breeze-custom.rpm
```

At the end of the installation it should prompt you to reboot, which you have to do to apply the changes.

10. Select the theme
After rebooting, you can open the `Login Screen (SDDM)` app to see a list of installed themes. Your theme should appear here!
