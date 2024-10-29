---
title: "VSCode in Podman dev container" # in any language you want
cover:
  image: "images/posts/vscode-podman/cover.jpg"
  relative: false # To use relative path for cover image, used in hugo Page-bundles
description: >
    A guide on using VSCode in a Podman devcontainer (toolbox)
showToc: true
tocOpen: true
author: LM. Garret
UseHugoToc:
---

I wrote this guide since I installed a Fedora immutable desktop spin (Bazzite) on my desktop and decided to set it up for development as well. Since toolbox and Podman are already installed out of the box, I thought it would be interesting to go all the way in and develop using VSCode attached to a Podman devcontainer created by toolbox.
This unfortunately does not work perfectly out of the box, and I intend to keep this guide up-to-date with all the changes I had to perform in order to get my dev environment up and running!

## Using Podman instead of Docker
First, to make sure we can use Podman in place of Docker for dev containers, we need to adjust our VSCode settings as [mentioned in the documentation](https://code.visualstudio.com/remote/advancedcontainers/docker-options#_podman), to point to our Podman installation. In you VSCode settings file, add the following entries:

```json
  "dev.containers.dockerPath": "podman",
  "dev.containers.dockerSocketPath": "/run/user/1000/podman/podman.sock"
```

## Attaching VSCode to a running Podman container as a non-root user
There are multiple reasons why you wouldn't want to use the `root` user in your dev container: 
 - you have multiple shell configs for your non-root host user you would like to use in the container (e.g. getting the same experience as `toolbox enter my-container`)
 - you want to reuse your git config and credentials and do not want to have a distinct config per container
 - it feels wrong to use `root`! It's so often described as being a bad practice (which it is!), that even in a container we shouldn't have to default to using the `root` user

Now if you decide that you don't mind, then good for you! You can skip this guide's section :)

### 1. Container-specific config
If not already done, create your dev container with toolbox/any other tool of your liking. We will be referring to the container's name as the env variable `$TOOLBOX_CONTAINER_NAME` in the rest of this guide .

We will now need to create a specific configuration file for the container, to tell VSCode to attach to it as a different user, but also pass other environment variables from the host. For this file, I largely copied content from [this helpful GitHub issue's comment](https://github.com/containers/toolbox/issues/610#issuecomment-726057756), but don't hesitate to adapt it to your needs.

> Note: I chose to attach as my host's non-root user, hence passing the `$USER` env var. This comes with the added benefit of keeping my `.zshrc` and `.oh-my-zsh configuration` working out of the box and not having to create a different user in the container image and mount these files there.
> But please note that the user you will be using in the container has the same `$HOME` directory and that changes in the container **could reflect in your host user's home** ⚠️.


So open up the file
```command
vim "$HOME/.config/Code/User/globalStorage/ms-vscode-remote.remote-containers/nameConfigs/${TOOLBOX_CONTAINER_NAME}.json"
```

And write this content in it:
```json
{ 
  "remoteUser": "${localEnv:USER}",     
  "remoteEnv": {    
    "COLORTERM": "${localEnv:COLORTERM}",
    "DBUS_SESSION_BUS_ADDRESS": "${localEnv:DBUS_SESSION_BUS_ADDRESS}",
    "DESKTOP_SESSION": "${localEnv:DESKTOP_SESSION}",
    "DISPLAY": "${localEnv:DISPLAY}",
    "LANG": "${localEnv:LANG}",
    "SHELL": "${localEnv:SHELL}",
    "SSH_AUTH_SOCK": "${localEnv:SSH_AUTH_SOCK}",
    "TERM": "${localEnv:TERM}",
    "VTE_VERSION": "${localEnv:VTE_VERSION}",
    "XDG_CURRENT_DESKTOP": "${localEnv:XDG_CURRENT_DESKTOP}",
    "XDG_DATA_DIRS": "${localEnv:XDG_DATA_DIRS}",
    "XDG_MENU_PREFIX": "${localEnv:XDG_MENU_PREFIX}",
    "XDG_RUNTIME_DIR": "${localEnv:XDG_RUNTIME_DIR}",
    "XDG_SESSION_DESKTOP": "${localEnv:XDG_SESSION_DESKTOP}",
    "XDG_SESSION_TYPE": "${localEnv:XDG_SESSION_TYPE}"
  },
  "settings": {
    "dev.containers.copyGitConfig": false,
    "dev.containers.gitCredentialHelperConfigLocation": "none"
  }
}
```

### 2. Allowing non-root user to install VSCode's server
Here we face one of the major problems when trying to set-up this developer environment. Upon attaching to the container, VSCode will try to install its `.vscode-server` to the directory pointed out by `$HOME`. But:
 - Podman [apparently sets it to `/root`](https://github.com/microsoft/vscode-remote-release/issues/7657#issuecomment-1441350625), misleading the dev containers extension
 - We don't want to install the server into our host's home directory, otherwise it could conflict between dev containers

A workaround here is simply to allow your non-root user to modify file in the `root` user's home directory, `/root`. This would allow VSCode to create the `/root/.vscode-server` when attaching

> Note: the cleaner approach to perfectly isolate your container user from your host machine's home directory would be to create a different user (e.g. `vscode`) in the container's image, with a distinct home directory (e.g. `/vscode`). We could then use this user as the `remoteUser`. As mentioned earlier, in my setup I wanted a more seamless experience that comes with the pitfall of a shared home directory. If you follow this guide, make sure to understand when developing what commands could impact your home directory!

To give the write permissions to our home user, we will be using FACLs. First, login as root:
```command
su - root
```

then allow your user to write to the `/root` directory

```command
setfacl -R -m u:myuser:rwx /root
```
replacing `myuser` with your username. You can check with `getfacl` that the user now has the write permission on the `/root` directory

```command
getfacl /root
# file: root
# owner: root
# group: root
user::r-x
user:myuser:rwx
group::r-x
other::---
```

> Note ⚠️: **never do this on your host machine**! In our scenario, it is fine since we can login as root in the container without password anyway, so an attacker wouldn't even need to log as your non-root user to try to escalate privileges. But on your host machine, it's a different story and you shouldn't never mess with root files permissions!

### 3. Attach VSCode to the running container
You can now try attaching VSCode to your running container. It should install the VSCode remote server as expected, and you should be able to browse your project's files. Also, opening a new integrated terminal should be done using your non-root user!

### 4. (Optional) Fix user not in sudoers
It may be that your non-root user is not in the sudoers file. Instructions will depend on the container's distribution your using, but for Fedora user you can simply follow these steps:

1. Login as root
```command
su - root
```

2. Add `myuser` to the wheel group
```command
usermod -a -G wheel myuser 
```

3. Reload group
```command
newgrp wheel
```

4. Logout of the `root` user (usually Ctrl+D) and check that you're now a sudoer:
```command
sudo echo "It worked :)"
It worked :)
```

## Other tweaks
As I started using this new dev environment, I ran into several small problems that don't justify whole posts by themselves. But since others may run into the same issues, I decided to add them here just in case.

### Fix slow DNF command (and others) if using custom DNS
I'm a fairly privacy minded person, for example I used to work at Inpher on a privacy-preserving distributed computation framework. As many others, I tweaked my DNS settings to use a different provider, [NextDNS](https://nextdns.io/) in my case. Unexpectedly, it comes with a few side-effects, including [a slow-startup for Firefox](https://discussion.fedoraproject.org/t/firefox-takes-several-minutes-to-open/114320). For this issue,
a workaround is to make sure to have:
```conf
127.0.0.1   localhost fedora bazzite
```
in `/etc/hosts`, so that the domain resolution for these take not time and does not hang the app's startup.

This side effect is unfortunately also visible in `toolbox` containers, where `dnf` commands can take a few minutes to actually start printing anything. Again, the fix here is to make sure to have `toolbox` point to `127.0.0.1` in your **host machine**'s `/etc/hosts` file, so:
```conf
127.0.0.1   localhost fedora bazzite toolbox
```

`dnf` is now as fast as I could expect in `toolbox` containers!

### Setting GPG to sign commits and tags
I recently started signing my commits and tags with GPG, increasing the security guarantees of the projects I'm working on. So naturally, I wanted to sign the commits done inside the `toolbox` dev containers I'm using. These changes may be very Fedora KDE/Bazzite specific!

#### Set a pinentry program for GPG
First, to create or use a GPG key, we need to set the `pinentry-program` used to prompt us for our passphrase. On the host machine, edit the `~/.gnupg/gpg-agent.conf` file
```command
vim ~/.gnupg/gpg-agent.conf
```

and set the content to 

```
pinentry-program /usr/bin/pinentry-qt
```

> From the Archlinux doc: The pinentry programs [...], /usr/bin/pinentry-qt, [...] support the DBus Secret Service API, which allows for remembering passwords via a compliant manager such as GNOME Keyring, KeePassXC or KDE Wallet.

Reload the agent with
```command
gpg-connect-agent reloadagent /bye
OK
```

And try creating a key with
```command
gpg --full-generate-key
```

or test signing a message with
```command
echo test | gpg --clear-sign
```

If it doesn't work, try killing the agent then reloading it:

```command
pkill gpg-agent
```

..and that's pretty much it! VSCode in the dev container will now open the `pinentry-qt` program to ask you for your passphrase when committing/tagging commits.