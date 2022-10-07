---
date: "2014-12-18T00:00:00Z"
title: 's6: Simple init system'
aliases:
- post/2014-12-18-s6-simple-init-system
---

One of the natural consequences of using [lightweight Docker hosts](/2014/12/17/dock0-round-2) and running everything in containers is that I tend not to interact with the actual VM's system as much. I SSH straight to a container for IRC, I do my kernel builds in containers, etc. With this, it's made sense to strip out components out of the VM that were used solely for user interaction.

Some of those components were easy to identify and pull out:

* Stopped cloning my dotfiles and scripts repos to the VM image
* Removed packages like vim-minimal and zsh

But the big change I wanted to make was simplifying the init process itself.<!--more--> Don't be fooled: I love systemd (from a user perspective), and for full systems that I'll be interacting with I love the control systemctl provides. But for a system that will only be running a handful of daemons, and that I won't ever be changing other than to update the static assets and reboot, systemd is massively overkill.

I've worked with several lightweight process managers as part of my container building. For within the containers, I initially chose [runit](http://smarden.org/runit/). I eventually switched to [s6](http://www.skarnet.org/software/s6/), which offered a more expansive array of helper tools and had a bit better support for signal handling and early/late init control. Based on that knowledge, I decided to use s6. The s6 site has [a great post](http://www.skarnet.org/software/s6/s6-svscan-1.html) on how to use s6 as your pid 1, and I referred to it continuously for guidance.

As per the post, the work done by pid1 is split between 3 main "stages":

Early init
----------

This stage starts when the initrd hands off control of the system to the rootfs. It moves the newly mounted root filesystem to `/` and execs your init process. The point of this stage is to set up the basic system to the point that service management can begin. When I started, I expected this to be pretty straightforward: make sure /proc, /sys, and friends are mounted and move on. I was mostly right, and I ended up with the following stage1 script:

{{< gist akerl 34196f0a07ed70b98942 >}}

The script uses [execline](http://skarnet.org/software/execline/), which is essentially a strict cousin of bash with more rigorous handling of conditionals and command parsing.

This script does support "tasks", which I added during development to handle things that need to happen once during startup. Originally, I wanted these to run in the foreground and block, but that leads to interesting issues when tasks and services need to cooperate. For example, [one of the tasks](https://github.com/dock0/rootfs/blob/master/overlay/etc/s6/task/pacman-init) initializes pacman's keys. That needs entropy, which doesn't exist in sufficient quantities until the haveged service starts. To resolve this, I switched tasks to run in the background and decided I'd handle checking for any dependencies in the task / service files themselves to prevent race conditions.

I also learned about [process groups](https://en.wikipedia.org/wiki/Process_group) when I first booted and discovered that pid1 didn't appear in htop. It turns out that htop hides kernel threads by default, and if you don't do anything, s6 is still in group 0, belonging to the kernel. Running `s6-setsid` sets the process group to match the current PID, which in this case is 1.

Once that's done, we load the environment and start svscan, moving us to stage 2.

Normal operation
----------------

This stage is pretty boring, which is intentional. s6-svscan is dead simple to understand: it watches a directory, and spawns an s6-supervise process for any directories inside. The s6-supervise process runs `./{directory}/run`, which should run whatever the service is in the foreground. If the process dies, s6-supervise restarts it. If you want to stop it, just tell s6 to stop it. The run scripts thus end up looking super simple:

{{< gist akerl dcee8549a65cfb5b5ae3 >}}

The primary issue here is making sure all your processes run in the foreground, but everything I've wanted to run thus far has either defaulted to that or had an easy flag for foreground operation.

This mode keeps going until the system is shut down or melted down into raw materials, at which point s6-svcan runs ./.s6-svscan/finish, which I've symlinked to my stage3 script.

Graceful self-destruct
---------------------

Stage 3 in my case is way simpler than most systems. I'm not saving any state, and all the system bits are read-only artifacts, so there's not much fear of inconsistent writes or shutting processes down cleanly.

That said, my main issue was getting the system to enter stage 3 at all. Crazy, right? Hitting shutdown in Linode's UI wasn't successfully triggering stage 3, it was timing out after 2 minutes and the host was forcibly pulling the plug on my VM. My initial hunch was that the shutdown signal wasn't being properly handled by s6, but I confirmed that manually sending SIGINT to pid 1 caused the shutdown to trigger as expected. I then moved on to investigating how a shutdown request from Xen translates to a SIGINT on pid 1. Essentially, Xen sends a hypercall to the VM which translates roughly to a ctrl-alt-delete, and the kernel reacts to that. I searched some more and found /proc`/sys/kernel/ctrl-alt-del`, which controls whether the kernel tries a graceful shutdown via SIGINT or a forcible one. That ended up being a dead end, since neither option successfully shut down the system, but it did lead me to hunting around a bit more in /proc. While doing so, I found `/proc/sys/kernel/poweroff_cmd`, which defaults to `/sbin/poweroff`. I couldn't really find good docs for this, outside of reading the kernel code, but I tried setting it to the correct binary (/usr/bin/s6-reboot) and shutting down, which worked! Rather than modding that sysctl setting in the future, I set up symlinks so that /sbin/{halt,poweroff,shutdown} would all point to /usr/bin/s6-reboot.

Once that was sorted, the actual stage 3 code was easy:

{{< gist akerl f9dee1d69f9124c624e7 >}}

It reclaims control of the console for stdout/err, syncs the disks, kills the processes (first gracefully, then hard), waits for them to be reaped, then syncs again and reboots.

Final product
=============

The end result is a fully functional init system for my VM:

{{< gist akerl f4d6f57c97b2c44b8df8 >}}

There were some definite "gotcha" moments while building it, and overall I feel like it was very educational. Things like "wow, I guess I have to write something to load /etc/hostname into the running system" and having to template out my own network configs rather than leaning on a larger tool were very eye-opening experiences, and it gives a great perspective on the things most folks take for granted about their operating system.

There are still plenty of things to work on, from little tweaks (like ensuring lvmetad is running) to big projects (shipping system logs to a centralized service), and I'm excited to dive into them.

