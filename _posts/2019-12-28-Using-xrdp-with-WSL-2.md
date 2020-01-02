---
layout: post
title:  "Using xrdp with WSL 2"
author: David Dhuyvetter
tags: [Windows, Linux, WSL]
excerpt: The Windows Subsystem for Linux is in the midst of a complete overhaul. I've used it in the past for development tooling, and while it had strengths from the start, I always seemed to hit hard limitations that made it more trouble than it was worth.  Initially I used VcXsrv for GUI applications, and while I was able to get some productivity out of the configuration, it was a lot of work to set up and maintain.
---

This is part 1 of a 2 part journey.  I'm writing it up here to preserve the details, but this isn't the configuration that ultimately worked for me.  For that, you should see: [Using xpra with WSL2](2020-01-01-Using-xpra-with-WSL-2.md).

The [Windows Subsystem for Linux](https://docs.microsoft.com/en-us/windows/wsl/about) is in the midst of a complete overhaul. I've used it in the past for development tooling, and while it had strengths from the start, I always seemed to hit hard limitations that made it more trouble than it was worth.  Initially I used [VcXsrv](https://sourceforge.net/projects/vcxsrv/) for GUI applications, and while I was able to get some productivity out of the configuration, it was a lot of work to set up and maintain.

With [WSL 2](https://docs.microsoft.com/en-us/windows/wsl/wsl2-index), a real linux kernel is now running in a lightweight VM instead of WSL mapping system calls to the Windows kernel. Docker server, the ultimate "ain't never gonna work" under WSL 1, now works under WSL 2. But this new approach actually makes running GUI applications harder.  The network configuration is more complex. The IP address, and even the subnet used by WSL changes periodically. Setting up a secure X server in Windows would require SSH tunneling to WSL, or a [service](https://github.com/shayne/go-wsl2-host) to update the Windows hosts file whenever WSL changes its address, or custom Windows Firewall configuration or ... .

Rather than going down that rat hole, I wanted to look at other options.  Noticing that Hyper-V now supports [enhanced session mode with Ubuntu VMs](https://www.omgubuntu.co.uk/2018/09/hyper-v-ubuntu-1804-windows-integration) I wanted to see whether the same capabilities could be brought to WSL 2.

Most of this is based on [instructions from Robin Kretzschmar](https://dev.to/darksmile92/linux-on-windows-wsl-with-desktop-environment-via-rdp-522g). I've added a [working audio configuration from Griffon](https://c-nergy.be/blog/?p=12469).  

Start by [installing a WSL 2 Ubuntu distro](https://docs.microsoft.com/en-us/windows/wsl/wsl2-install).  Then fromthe Ubuntu bash prompt, configure the system:

```bash
# desktop
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install -y xfce4

# xrdp
$ sudo apt install -y xrdp
$ sudo cp /etc/xrdp/xrdp.ini /etc/xrdp/xrdp.ini.bak
$ sudo sed -i 's/3389/3390/g' /etc/xrdp/xrdp.ini
$ sudo sed -i 's/max_bpp=32/#max_bpp=32\nmax_bpp=128/g' /etc/xrdp/xrdp.ini
$ sudo sed -i 's/xserverbpp=24/#xserverbpp=24\nxserverbpp=128/g' /etc/xrdp/xrdp.ini

# firefox
$ sudo apt install -y firefox
$ sudo vi /etc/apt/sources.list # uncomment: deb http://archive.canonical.com/ubuntu bionic partner
$ sudo apt update
$ sudo apt install -y adobe-flashplugin browser-plugin-freshplayer-pepperflash

# audio
$ sudo apt install -y xrdp-pulseaudio-installer
$ sudo xrdp-build-pulse-modules # this will fail
$ sudo vi /etc/apt/sources.list # uncomment deb-src for bionic and bionic-updates
$ sudo apt update
$ cd /tmp
$ sudo apt source pulseaudio
$ cd pulseaudio*/
$ sudo ./configure
$ cd /usr/src/xrdp-pulseaudio-installer
$ sudo make PULSE_DIR="/tmp/pulseaudio-11.1"
$ sudo install -t "/var/lib/xrdp-pulseaudio-installer" -D -m 644 *.so
```
At this point exit the wsl shell and verify that the distro shuts down by calling `wsl -l -v` at a windows command prompt.  Then launch the Ubuntu command window again to continue.

```bash
$ sudo /etc/init.d/xrdp start
```

Now, from Windows, open `Remote Desktop Connection` and connect to `localhost:3390`  The user name should be your Ubuntu user name. Before connecting you can set the Display Configuration to `Full Screen`.  When you connect, you will be prompted for the Ubuntu user's password, and then the XFCE desktop will open.

**Notes**:

- works well with multiple monitors
- clipboard integration works at least for plain text
- noticeable CPU load with this configuration
- video playback is not smooth
- performance is noticeably slower than native
- audio/video sync is bad
- the remote session disconnects when the host computer sleeps, but can be reconnected.

One advantage of this configuration is that you are running a full blown XFCE Desktop. Programs that assume that they are running under a Desktop work well.

The biggest drawback is that the Linux desktop and the Windows desktop are not unified.  Running email/zoom/slack natively in Windows and doing development in Linux would end up being an endless series of switches in and out of the Linux desktop.

In the end, while this is better than running individual apps with a Windows XServer, it isn't the configuration that I'm looking for. In the next post I'll describe using xpra to integrate Linux apps running under WSL 2 with the Windows desktop.