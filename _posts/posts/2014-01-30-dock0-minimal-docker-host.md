---
layout: post
title: dock0: A minimal Docker host
---
Linux containers have recently been in the spotlight, in no small part due to [Docker's](https://www.docker.io) work in providing a user-friendly interface to [LXC](http://linuxcontainers.org/). Lightweight containers allow us to reimagine how we handle deployments, separation of services, and infrastructure management. A system that can be spun up in seconds, configured on the fly, and spun down just as fast presents the next step in a trend begun by the virtual private server.

In a world where all the action happens inside containers, it seemed wasteful for the "host" system to be a full-service Linux system of its own. I went on a mission to strip down an Arch system as far as I could, to the point where I had a read-only system with only essential services from which to run Docker.

<!--more-->

The [ArchISO](https://github.com/djgera/archiso) project was an excellent starting point: it lets you build an ISO with a read-only Arch system utilizing a SquashFS, and [Arch's docs](https://wiki.archlinux.org/index.php/Archiso) on using it are awesome. Going through their code, I identified the parts needed to create a read-only Arch system:

1. Make a file and format it as a filesystem (I chose ext4). Mount it somewhere; that's going to be the root filesystem of your new installation.
2. Use the `pacstrap` tool to install all the packages you want onto that filesystem.
3. Adjust the resulting filesystem as desired (you can use arch-chroot to hop inside the filesystem, or work on it directly from the existing system).
4. If you want Docker to be happy, make sure /var/lib/docker/devicemapper/devicemapper/data exists and is a symlink to another device besides the root filesystem. Docker needs to make its devices there, and if that file is on your read-only root device, it's gonna have a bad time.
3. Build a kernel for your new system. You'll want to make sure it supports SquashFS, and the [various Docker options](http://docs.docker.io/en/latest/installation/kernel/).
4. Build an initramfs for your kernel, and include the necessary scripts to find and mount the SquashFS. The ArchISO project includes a pretty involved script. I stripped it down to just what I needed: https://github.com/akerl/my_dock0/blob/master/initcpio/hooks/dock0
5. Squash up the root filesystem and drop it, the kernel, and the initrd onto the target disk.

Because I didn't plan on doing all that manually, I wrote [dock0](https://github.com/akerl/dock0). It automates the above steps, using a provided configuration. Because I couldn't think of a more clever name, I called my configurations [my_dock0](https://github.com/akerl/my_dock0). The dock0 tool supports stackable configuration files and dynamic scripting so that you can really use it for crafting just about any kind of Arch image. My configuration makes use of [roller](https://github.com/akerl/roller) to build the kernel, even though it feels a bit dirty to have Ruby shell out and run Python.

Since I'm using this tool on my Linodes, I realized that even the initial system configuration could be automated. I wrote a [StackScript](https://www.linode.com/stackscripts/view/?StackScriptID=8125) that uses dock0 to convert the attached disk images into my desired host platoform, and then I wrote a [deploy script](https://github.com/akerl/my_dock0/blob/master/meta/deploy.rb) to run locally and deploy the StackScript via Linode's API. Using this, I can run a single command locally, which will rebuild the given Linode into a new dock0 host running Docker.

Having done this, I'm now beginning the process of converting my services into Docker containers. That effort will hopefully get its own blog post one day.
