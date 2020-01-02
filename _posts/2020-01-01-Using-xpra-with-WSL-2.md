---
layout: post
title:  "Using xpra with WSL 2"
author: David Dhuyvetter
tags: [Windows, Linux, WSL]
excerpt: Xpra provides a server side session for X11 applications, supports audio and clipboard integration, and allows the Linux apps to use the Windows taskbar and system tray.  Adding xpra to WSL 2 allows Linux apps to integrate well in the Windows desktop
---

This is part 2 of a 2 part journey.  Initially [I looked at xrdp](2019-12-28-Using-xrdp-with-WSL-2.md), which has reasonable performance, works well with multiple monitors, supports audio and clipboard, and maintains window state on the server side allowing clients to disconnect and reconnect.  However, with xrdp there is noticeable CPU load, you have to explicitly reconnect the client after host sleep, and the Linux applications are all in a distinct desktop from the Windows applications.

[Xpra](https://xpra.org/) provides a server side session for X11 applications, supports audio and clipboard integration, and allows the Linux apps to use the Windows taskbar and system tray.  Adding xpra to WSL 2 allows Linux apps to integrate well in the Windows desktop

## But ... why?

It is a valid question.  I am ultimately looking for a configuration that would be best provided by just skipping the whole Window/WSL path and installing Linux.  My current laptop is running [Linux Mint](https://linuxmint.com/) and it works fine.  The multi-monitor HiDPI configuration was painful and entirely manual to set up, but with [autorandr](https://manpages.ubuntu.com/manpages/bionic/man1/autorandr.1.html) I can now move my laptop from workstation to workstation, and all the monitors work with good scaling.

With the exception of Visual Studio (which I use infrequently) everything I need runs, either natively in Linux, or as a web app in Chrome.  If it weren't for the fact that I can't get on the corporate WiFi, and other issues that are more about policy than technology, I would be happy to stay where I am.

At home, I use a shared Laptop, and as much as I would like to convince everyone else to use Linux, that isn't going to happen.  I've dual-booted before, and found that even with cross mounted file systems, however the system was booted up, the tool/data/login I needed was always a reboot away.

If I didn't have these constraints, I would absolutely be sticking with Linux and not bothering with Windows/WSL.  As much as my co-worker likes to get in digs about "the year of the linux desktop" Linux has been a fine desktop OS (for a technically inclined person) for many years now.

## Setting up WSL 2 with xpra

WSL 2 is currently available for Windows 10 in [Windows Insiders](https://insider.windows.com/en-us/) in the slow ring.  It is targeted for general release in the [20H1](https://www.howtogeek.com/438830/whats-new-in-windows-10s-20h1-update-arriving-spring-2020/) update which will be released some time in the spring.  It is built on top of [Hyper-V](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/about/).  If you are running Windows 10 Home, Hyper-V is not available, but WSL 2 is. Microsoft has refactored its virtual machine platform to allow delivery of WSL 2 without the full Hyper-V product.

The [instructions for installing WSL 2](https://docs.microsoft.com/en-us/windows/wsl/wsl2-install) are pretty straightforward.  Converting a distro from WSL 1 to WSL 2 is very time consuming, so it is worth while to set the default WSL version to 2 _before_ installing Ubuntu.

```cmd
c:\> wsl --set-default-version 2
```

Once the WSL default is set to 2, install [Ubuntu](https://www.microsoft.com/en-us/p/ubuntu/9nblggh4msv6) or [Ubuntu 18.04 LTS](https://www.microsoft.com/en-us/p/ubuntu-1804-lts/9n9tngvndl3q) from the Microsoft Store.

The **most important** thing to know about installing xpra, is not to install the version in the default repository.  It is old and broken, and will leave you thinking that xpra doesn't work at all.  Instead, install from the xpra repository.

```bash
# desktop
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install -y xfce4

# xpra
$ sudo wget -O /etc/apt/sources.list.d/xpra.list https://xpra.org/repos/bionic/xpra.list
$ wget -qO - https://xpra.org/gpg.asc | sudo apt-key add -
$ sudo apt update
$ sudo apt install -y xpra
$ sudo apt install -y libgstreamer1.0-0 gstreamer1.0-plugins-base gstreamer1.0-plugins-good gstreamer1.0-plugins-bad gstreamer1.0-plugins-ugly gstreamer1.0-libav gstreamer1.0-doc gstreamer1.0-tools gstreamer1.0-x gstreamer1.0-alsa gstreamer1.0-gl gstreamer1.0-gtk3 gstreamer1.0-qt5 gstreamer1.0-pulseaudio
$ sudo usermod -a -G xpra <your WSL login>

# Firefox
$ sudo apt install -y firefox
$ sudo sed -i '/^# deb http:\/\/archive.canonical.com\/ubuntu bionic partner/s/^# //' /etc/apt/sources.list
$ sudo apt update
$ sudo apt install -y adobe-flashplugin browser-plugin-freshplayer-pepperflash

#launcher
$ wget -qO - https://build.opensuse.org/projects/home:manuelschneid3r/public_key | sudo apt-key add -
$ $ echo 'deb http://download.opensuse.org/repositories/home:/manuelschneid3r/xUbuntu_18.04/ /' | sudo tee -a /etc/apt/sources.list.d/home:manuelschneid3r.list
$ sudo apt install -y albert
```
The [Albert](https://albertlauncher.github.io/) launcher is a great single app to start with xpra.  It sits in the Windows 10 system tray and allows you to start other programs.  I have also tried [Ulauncher](https://ulauncher.io/), but it seems less happy than Albert to run without a full blown Linux desktop

Albert uses the `*.desktop` files in `/usr/share/applications`, but some of these files need to be modified to be used in this configuration.

For example, to allow Albert to launch gnome-terminal

```bash
$ mkdir ~/.local/share/applications
$ grep -v OnlyShowIn /usr/share/applications/gnome-terminal.desktop > ~/.local/share/applications/gnome-terminal.desktop
```

Once the install is done, exit the bash shell, and verify that WSL is shut down by calling `wsl -l -v` in a windows command prompt.

```command
C:>wsl -l -v
  NAME            STATE           VERSION
* Ubuntu          Stopped         2
```

With that, you can re-open the bash shell and start the xpra server with the following script:

```bash
#!/bin/bash

sudo rm -rf /run/xpra
sudo install -m 0775 -g xpra -d /run/xpra
export XDG_RUNTIME_DIR=/run/xpra
xpra start --start='/etc/X11/Xsession albert' --bind-tcp=0.0.0.0:14500 --speaker-codec=wav --microphone-codec=wav
```

This script is a bit of a hack to work around issues caused by the quirks of the WSL setup.  It starts xpra as a daemon, so once it is running, you can exit the WSL bash window.  The WSL VM will not shutdown because of the running daemon.

## Host client configuration

Download the 64 bit Windows client installer from [xpra.org](https://xpra.org/) and install it.  Create a shortcut to launch it.   The `Target` should be `"C:\Program Files\Xpra\Xpra.exe" attach tcp://localhost:14500 --speaker=off --microphone=off --opengl=no` and it should `Start in`: `"C:\Program Files\Xpra"`

The client takes advantage of the fact that listeners in WSL are mapped to localhost in Windows.  It is launched with speakers and microphone off, because having these on results in a ~4% CPU load at all times.  Don't worry, they can be turned on if/when needed.

OpenGL is disabled because it is unstable (at least on my system) and results in frequent client crashes.  The good news is that other than [glxgears](https://www.x.org/archive/X11R7.0/doc/html/glxgears.1.html) I haven't found any applications that suffer from OpenGL being off.  If you were hoping to play lots of games in WSL 2, YMMV.

Double click on the shortcut, and in a couple of seconds, the Albert welcome screen should appear on your screen.

![Albert welcome screen](/images/using-xpra-with-wsl-2/Albert.png)

Make your choice, and configure Albert.

![Albert Configuration](/images/using-xpra-with-wsl-2/Albert2.png)

The Hotkey that you set will be usable when the focus is in a WSL X window.  You probably want to disable `Display shadow`, because it looks odd on the Windows desktop, and if you have multiple monitors, you probably want to disable `Always center Albert` because it thinks that the "center" is the space between the monitors.

On the `Extensions` tab, you should enable at least `Applications`.  With that, you can close the settings.  the Albert icon should remain in the Windows system tray.

![Albert ready to help](/images/using-xpra-with-wsl-2/Albert4.png)

If there are no open WSL X windows, simply click on albert in the system tray to bring it up.  The first time Albert is activated after launching xpra, it might not accept input.  If that happens, click the icon to close and re-open it and you should be good.

![Using Albert](/images/using-xpra-with-wsl-2/Albert3.png)

## Notes

- Performance is _very_ good. 
- As as long as audio is turned off, there is no constant CPU load just to have windows open.
- Audio quality is good, but there are occasional pops and it probably isn't good enough for music.
- Video quality is good even at full screen.
- Audio/video sync is off by several seconds.
- Windows stay open across host system sleep.
- The first time gnome-terminal is opened after launching xpra it might not accept focus.
- The first time apps launch after starting xpra they take a few seconds to get going. Subsequent launches are quick.
- [JetBrains Rider](https://www.jetbrains.com/rider/) has some odd quirks with focus and double click.  This may be a generic Java GUI issue.
- [VS Code](https://code.visualstudio.com/) works fine, but the wrong icon shows up in the Windows task bar.  I would swear that the right icon was there initially.
- Android Studio installs, but you can't run the emulator in WSL 2 because there is no pass-through emulation. 
- Works well with multiple monitors, but application splash screens often span the 2 monitors.
- I've seen hard (server side) crashes when plugging in and removing external monitors on the host ... but these _seem_ to have gone away with the latest build of the client.

## Conclusion

This isn't a perfect solution.   I am most bothered by the issues with Rider, since I use that tool frequently. But there really isn't a deal breaker.  The performance and CPU load are the best I've seen, and the fact that Windows stay up across host sleep is the killer feature.