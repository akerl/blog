---
layout: post
title: "systemd Services Are Easy"
---

I spend a decent amount of time thinking about init systems. Most of the time, that means [s6](http://skarnet.org/software/s6/), but for more complex or user-interactive systems, I'd go for systemd. This puts me squarely opposed with a decent-sized group of loud people, it seems. One of the complaints that is occasionally brought up is that sysvinit was great and init scripts are great, etc etc. My hypothesis: for people whose primary init system interaction is writing and using initscripts, systemd unit files are so amazingly easier to read and write and use that it is without a doubt the better choice.

<!--more-->

Let's make a systemd service
===========

Services for systemd are started using "unit files". The full spec for unit files is long and boring. I'll link it at the bottom, but please don't skip ahead and read it: doing so is like reading the dictionary to learn English.

We're going to write a unit file for nginx. Let's start our unit file off by saying what it's for:

{% gist akerl/a95f4a3b51f349173f91bd617955f15c %}

It's just a text file, named "nginx.service". The ".service" calls it out as a service file (there are some other types, which aren't immediately relevant: timers for cron-like behavior, targets for handling more meta interactions between services, etc). The unit file uses the INI format, and the first section we've added is "Unit", which calls out some information about the unit file itself and how systemd should think about it. We've given the description, which will show up if somebody asks systemd about this service.

Now lets tell it how to start the service:

{% gist akerl/e5f24d51290c4157a414416675a26526 %}

We've added a "Service" section, that tells systemd how to interact with the service, and an "ExecStart" setting. The default expectation of systemd is that the process will run in the foreground (without forking), so we tell nginx not to fork. But it's pretty annoying to have to tell all your daemons not to fork, and one of systemd's key features is using cgroups at the kernel level to track processes, so we can tweak the unit file to be smarter:

{% gist akerl/f9e2fbb645cabc4d3d673b6c3f08e458 %}

We tell it the type of service is "forking", which means it should expect the initial process to fork and exit. Additioanlly, we can tell it where the service writes its PID file. That's not required (systemd is tracking the children using cgroups to know if nginx is still running), but it lets systemd know which nginx child process nginx thinks is the main one, so that its status output can be more helpful.

So now you have a working unit file. What do you do with it? User-created systemd unit files go into /etc/systemd/system, and then you link them into the "target" you want to launch them. Targets roughly equate with "the state I want my system to be in". So for example, the standard target for "start everything the system should normally run" is "multi-user.target". It has a cooresponding directory, "/etc/systemd/system/multi-user.target.wants". To enable your service for that target, just symlink it in:

{% gist akerl/542156d0f5ab5c6104c83510a662b4cb %}

Now you can do "systemctl status nginx" and see its status, and likewise for start/stop/restart/etc. By default, systemd assumes that it should kill the running processes to stop, and for restart it should kill all the processes and then run the ExecStart again. But you're smarter than that! You know that you can reload nginx just by sending it a SIGHUP! So let's make systemd smarter:

{% gist akerl/3bdf0657419bc64aa347cc3848d3679f %}

Awesome! Now it knows that 'systemctl reload nginx' should just SIGHUP the main PID. It knows the main PID based on the PIDFile setting you gave it already.

We also know that you can be smarter about stopping nginx by knowing the PID:

{% gist akerl/1c51ddb6b0101c8336b2c6b5bc371047 %}

So now we've taught it that it can used "mixed" mode to kill nginx. KillMode can be "none", "stop" won't kill anything. "control-group" (the default) will sigkill all the cgroup's processes. "process" will SIGTERM just the main PID, and mixed will do both "process" and "control-group". We've also told it to use SIGQUIT on the main PID, which nginx knows to handle gracefully.

The last missing piece that separates the above from a real live unit file is dependencies: you probably want nginx to come up after your network, for instance. Thankfully, telling systemd about dependencies between units is easy:

{% gist akerl/119dfaad8547b382c8b6a6e074dfdbcd %}

It'll now start after the network target. You can add as many Afters as you want, by space-separating them on the same "After=" line. You can also depend on specific services, if you know you need them.

Next steps
==========

So now you have a functional unit file. To keep me honest, here's [the real deal nginx unit file in Arch](https://git.archlinux.org/svntogit/packages.git/tree/trunk/service?h=packages/nginx). I left out a few fancier declarations around logging and device lockdown, but otherwise it's exactly what we built above.

If you want to explore fancier options, I highly recommend starting with [the ArchWiki guide to unit files](https://wiki.archlinux.org/index.php/Systemd#Writing_unit_files) and using [the index of directives](https://www.freedesktop.org/software/systemd/man/systemd.directives.html) as a reference guide.

If you think I'm totally wrong or I've been corrupted by system or some such, feel free to tweet angry messages at me.

