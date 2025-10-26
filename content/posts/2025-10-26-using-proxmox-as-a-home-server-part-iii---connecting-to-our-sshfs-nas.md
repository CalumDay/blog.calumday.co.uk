---
title: Using Proxmox as a Home Server Part III - NAS Connections
description: In part 3 of this series, we will discuss how to connect a range of devices to out SSHFS NAS.
date: 2025-10-26T16:51:57.955Z
preview: /media/posts/pve-rack/Drive_Cables.jpg
draft: true
tags:
    - Guide
    - Network
    - Projects
    - Proxmox
    - Server
    - Virtualisation
    - ZFS
    - LXC
categories: []
toc: true
autonumber: true
math: true
hideBackToTop: false
hidePagination: false
showTags: true
fmContentType: Typo Post
slug: proxmox-home-server-part-iii-nas-connections
keywords:
    - Container
    - Guide
    - Home Server
    - Proxmox
    - PVE
    - ZFS
---

Now that we've created the storage pool and spun up our container(s), we need to be able to connect our devices. There are a range of methods we can use to do this, and if theres a device thats not covered, there is almost certainly a guide out there for it.

## Linux Clients

First off, we start with Linux clients. We'll split this into 2 sections, temporary connections, and auto-mounts.

### Temporary Connections

Whether we're testing, or just want to quickly attach a device to download something before disconnecting again, we can easily connect to our NAS using a simple command. Before we can do so, we need to install the SSHFS package. On Fedora and related distros, we use the command:
```sh
sudo dnf install -y sshfs
```

We can then mount the share using the following command:
```sh
sshfs -o reconnect,ServerAliveInterval=5,ServerAliveCountMax=3,idmap=user "{user}@{ip_address}:/<PoolName>/<DatasetName>" "~/zpools/{DatasetName}"
```

This command mounts the remote location `/{PoolName}/{DatasetName}/` as `{user}` which in our example will be `root` on the server located at `{ip_address}` which is the local IP address we set in the previous chapter. it also uses a few additional options.

`reconnect` - Does what it says on the tin. It reconnected to the remote server if needed.
`ServerAliveInterval=10` - Checks the connection to the server every 10s. If there is no response, it assumes the connection is down.
`ServerAliveCountMax` - Sets the number of intervals in a row with no response before the connection is considered dead. When the threshold is reached, it disconnects.
`idmap=user` - Maps the local UID to the remote users UID which ensures that permission problems don't arise from miss-matched UIDs.

### Auto-Mounts

If we want a more permanent connection which automatically connects at login, systemd units are an excellent way to accomplish this. We first need to install SSHFS, using the first command in the previous section. Next we create a systemd user directory (unless it already exists):
```sh
mkdir -p ~/.config/systemd/user
```

Then we create the auto-mount unit:
```sh
nano "~/.config/systemd/user/home-{username}-zpools-{DatasetName}.mount
```

Its important that the section after the last slash replicates the local path where the NAS will be mounted, with dashes in place of slashes.

We then populate this file with:
```sh
[Unit]
Description=Mount SSHFS remote Dataset {DatasetName}
Wants=network-online.target

[Mount]
What={IP_Address}:/{PoolName}/{DatasetName}
Where=/home/{local_username}/zpools/{DatasetName}
Type=fuse.sshfs
Options=_netdev,reconnect,ServerAliveInterval=10,ServerAliveCountMax=3,idmap=user,x-systemd.automount
TimeoutSec=120

[Install]
WantedBy=default.target
```

We can now start the unit and enable it to ensure it mounts on boot.
```sh
systemctl --user daemon-reload && systemctl --user enable --now home-{username}-zpools-{DatasetName}.mount
```

This will start the mount at boot, but in order to actually mount it something will need to try to access it, at which point it will automatically mount, thanks to the `x-systemd.automount` set in the Options line.

## Apple TV

Apple TV also supports SSHFS mounts for media, though only through third party apps. The VLC media player app works, though I've found it is slow and can be a little buggy. Instead, the Infuse app works excellently, and is well worth the £10 a year.

There is one big caveat, you'll need to enable password based SSH root login on the virtual machine, as there is no support for SSH keys. This is an excellent reason to have multiple VMs with their own mountpoints, so that you can grant access to only the media portion of your NAS.

To connect, simply head to the in app Settings>Shares menu, and select "Other". Give it a name, select SFTP as the Protocol, and enter the IP address, username, and password for the container we created. It should connect, and begin to scan for available media.

## Windows Clients

Adding support for SSHFS to windows is easy enough. I havent used it since I finished my thesis, and had no more need of Windows (there were a few bits of software that were both very specialised, and only available for Windows, and I couldn't be bothered getting them to work with compatibility layers like Wine), but it worked well the few times I did need it.

We need the SSHFS-Win and WinFSP packages, which can be installed by running:
```sh
winget install SSHFS-Win.SSHFS-Win
winget install WinFSP.WinFSP
```

Once installed, there are 2 options for mounting remote drives. I haven't used the `net use` command, but instructions are available in the [SSHFS-Win github](https://github.com/winfsp/sshfs-win).

Using Windows Explorer is as simple as selecting "Map Network Drive" under "This PC" in Explorer, and then entering the path using the following syntax:
```sh
\\sshfs\{Username}@{IP_Address}\{PoolName}\{DatasetName}
```

The first time you connect it will ask for your password and username, however if you want to use an SSH key, we need to slightly change the command. By replacing `\\sshfs\` with `\\sskfs.k` and uses the key stored in `%USERPROFILE%/.ssh/id_rsa on the Windows system.

## MacOS Clients

I don't currently have access to a MacOS device to test, as awesome as the new ARM MacBooks are, but it should be possible to connect using the macFUSE and SSHFS packages. I can't give a more in depth guide myself without testing it, so instead I suggest searching for another guide.

While this isn't an exhaustive list of possible clients by any means, its simply a list of the clients I have either used, or been asked how to use.

In the next installment of this series, we will start to look at hosting other services, specifically Home Assistant and ESP-Home.