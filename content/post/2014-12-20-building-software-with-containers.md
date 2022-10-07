---
date: "2014-12-20T00:00:00Z"
title: Building software with containers
aliases:
- /post/2014-12-20-building-software-with-containers
---

It seems like every day a new project is released for managing datacenters full of containers, all networked together and serving content to users. I enjoy that aspect of containers as much as the next sysadmin, but I've found one of the coolest use cases for them to be repeatable/isolated software builds.

Over time I've collected a decent list of codebases I want to utilize, and in the past I would pull and build them on the systems I planned to use them on. I've already talked at length about how poorly that scales, but now I'd like to focus on one specific area of the solution: using Docker containers to perform and share compiled software packages.

<!--more-->

The build process
=================

My build process has gone through several iterations, but from the start I had a few major goals:

* The compiled packages should be pushed somewhere public and easy to get to
* I wanted to use containers but not *require* containers: the build process should allow building outside of containers
* It should take care of all the dependencies for me

While I was brainstorming, [Jon Chen](https://github.com/fly) suggested that I use GitHub's [Releases](https://github.blog/2013-07-02-release-your-software/). It turned out to be a great idea, but I wanted a clean way to push up assets from the command line. Thankfully, I'd already written [octoauth](https://github.com/akerl/octoauth), a simple wrapper for handling GitHub API tokens. I set to work on a new project, called [targit](https://github.com/akerl/targit), to take care of assets for me. With it, uploading assets is as simple as `targit USER/REPO TAG /path/to/file`.

The build process is kicked off via a Makefile:

{{< gist akerl bd5aec507f54044e7c4d >}}

And the actual launch script called by the "container" target sets the container up with access to this repo, my SSH agent socket, and my git config:

{{< gist akerl e5cabf1f759b9f05bee9 >}}

The default command for dock0/build is `make local`, so when you run `make`, the following happens:

1. If you've not already checked out the upstream package's code, it updates the submodule
1. The Docker container is launched, with access to the right files
1. Inside the container, `make local` runs
1. Make runs the build target, which builds the repo
1. Make runs the push target, which tags this repo (not the upstream repo) and pushes the compiled package up to GitHub

If you don't want to use containers, running `make local` directly will perform similarly, though you'd need to make sure you have the dependencies yourself.

The great thing about this process is that it's very package-agnostic. The Makefile usually needs a few tweaks in the build step, but the overall pattern means I waste far less time remembering how to build whichever package I'm looking to update. And because the builds happen in containers, I spend less time chasing down dependencies or dealing with conflicts due to other things I've done on the system. I'm guaranteed a pristine environment for each build.

Challenges
==========

I ran into a couple weird hurdles while designing this process, partially because I was (am?) a Makefile newbie.

Versioning
----------

The first few things I used this process with were my own packages, so I got to make up my own version numbers. For those, I could just use some shell math to bump the patch number:

{{< gist akerl def541f4a82712f43a38 >}}

Obviously, for things that already have a version number, this doesn't work as well. I did want to track a patch number so I could tell one build from the next, so I settled on appending my patch number, and split that out into its own target:

([full Makefile here](https://github.com/amylum/s6/blob/master/Makefile))

{{< gist akerl 0a271bcf160fb218f7d7 >}}

Due to the way "thing = foo" variables are handled in Makefiles (recursively expanded every time they're used), the VERSION variable starts referencing the new value as soon as the ./version file is updated. This produces nice version strings that show both the upstream package's version and my patch number.

Tar in subdirs
--------------

I originally handled the tar command by using `-C build/`, which changes directories to build before running the tar. Unfortunately, doing this requires either hardcoding the list of files to use or using ".", because "\*" is expanded by the shell before tar is run. As such, you end up with a tar that includes ".". This is obnoxious when the tar is extracted, because depending on the user extracting it, it will try to change the ownership or permissions on ".", which for many packages is "/". No fun.

The solution for this was to switch to `cd build && tar`. I feel pretty unhappy about this, and I'm still hoping to find a cleaner solution.

What's next?
============

One downside to this is that I end up duplicating a lot of this. The Makefile and the ./meta/launch file end up being 80% identical between repos. I'm considering building some tool that abstracts the parts that don't change or provides default targets unless I override them.

