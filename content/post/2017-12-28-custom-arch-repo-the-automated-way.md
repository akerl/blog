---
date: "2017-12-28T00:00:00Z"
title: Custom Arch Repo the Automated Way
description: Automating a package repository for custom Archlinux images
aliases:
- /post/2017-12-28-custom-arch-repo-the-automated-way
---

At some point while working [VM images](/2014/12/17/dock0-round-2/) and [containers](/2014/12/20/building-software-with-containers/), I ended up wanting some custom Arch packages. It started with a desire for lighter packages and to really understand what was going into my system, and then turned into something of an obsession. As of today, I publish 92 Archlinux packages, most of them custom builds of common Linux tools. And because otherwise I'd be drowning in manual work, I've automated the hell out of the process.

<!--more-->

A high level overview
===========

First, a quick primer on the moving parts, starting backwards. We'll use the `git` package as an example:

1. <https://repo.scrtybybscrty.org/> is the usable repo that I put in my Archlinux images. It's got the git package listed
2. <https://github.com/amylum/repo/blob/master/git-amylum/PKGBUILD> is the "PKGBUILD", which is how Archlinux's package manager describes a package
3. <https://github.com/amylum/git/releases> has a list of my custom tarballs of pre-compiled (but not Arch-specific) git builds
4. <https://github.com/amylum/git/blob/master/.pkgforge> is the "magic" file that describes how to build those custom tarballs

Step 4: PkgForge files
==========

I originally build tarballs with a really long boring Makefile that I copied all over the place. It was ugly (50% my fault, 50% Make's fault), it didn't have great support for things with multiple dependencies, and it was hard to spot customization per-package since it was comingled w/ standard logic. In a burst of inspiration/madness, I wrote [pkgforge](https://github.com/akerl/pkgforge).

It has docs on how exactly .pkgforge files work, but the quick version is that they're very similar to [Homebrew's](https://github.com/homebrew/brew) formulas. They provide lots of scaffolding so that each file can contain just package-specific info. In the case of [our git .pkgforge linked above](https://github.com/amylum/git/blob/master/.pkgforge), we give it a name, some deps, and write out git's super complex `make` command.

Step 3: Making GitHub release artifacts
===========

As for actually running this file, we're all about automation, so it runs [here, on CircleCI](https://circleci.com/gh/amylum/git). The `circle.yml` file in the repo runs `make` and then `make release`. That has some scaffolding that you can see [here](https://github.com/amylum/pkgforge-helper/blob/master/Makefile), but the gist is this:

{{< gist akerl b98b1a0589bf7242134e279ffb7ddad0 >}}

So locally, when a new version of git comes out, I update the git submodule in my repo to point to the new version, commit that change, tag it with the new version number, and push. CircleCI then builds the new version and uploads it as a GitHub release artifact.

Step 2: Building Arch packages
===========

Since we've already compiled git, turning it into an Archlinux package is pretty easy. The [PKGBUILD file](https://github.com/amylum/repo/blob/master/git-amylum/PKGBUILD) just has to extract the tarball and move the license to the right place.

This phase runs in [a CircleCI build](https://circleci.com/gh/amylum/repo) as well, and uses [s3repo](https://github.com/amylum/s3repo), a tiny tool I wrote that basically just wraps Archlinux's package tooling to make it easier to build packages and toss them into S3, where I want to serve them from.

The [Makefile](https://github.com/amylum/repo/blob/master/Makefile) for the repo is potentially worth its own blog post, but for our purposes, all that's important to know:

* It calls `s3repo build` for any changed packages
* s3repo builds the package and signs it with a GPG key
* It then calls `s3repo upload` for each of those packages
* s3repo uploads the package and the signature to an S3 bucket
* s3repo then downloads the Archlinux repo's metadata file, adds these new packages to it, and uploads the new copy

Now we've got an S3 bucket with the package files, signatures, and metadata file to be an Archlinux repo.

Step 1: Have a usable repo
===========

To serve the repo, I use an AWS CloudFront distribution (basically, a CDN) in front of the S3 bucket. I could use the S3 bucket directly, but it doesn't support HTTPS. Using CloudFront lets me use AWS's certificate service.

This part is gonna sound weird now: this last bit isn't fully automated. I used terraform to define the components in [this repo](https://github.com/akerl/aws-account/blob/master/amylum/repo/main.tf), and the CircleCI config tests that the terraform is valid, but I run it manually using my own AWS creds. This is because so far I've not segregated out the creds that deploying this infra needs, and I don't want to give CircleCI administrator creds to my AWS account. Thankfully, this step only has to be done once when I first created the repo.

The result is that <https://repo.scrtybybscrty.org> serves up the packages from the bucket, and can be plopped directly into an Archlinux system's configuration, letting me start using my new custom packages.

Next steps
==========

* Commit/tag signing for my pushes to the GitHub repos for each package
* Signing of the artifacts when CircleCI builds them and uploads them to GitHub
* Segmenting the s3-website / repo code from my core aws-account repo. I semi-accidentally bundled a whole lot of loosely-related stuff there and have been meaning to break it out properly.
* Write a blog post on how I keep track of package updates upstream

