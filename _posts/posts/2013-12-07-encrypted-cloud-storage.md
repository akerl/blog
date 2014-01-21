---
layout: post
title: Encrypted Cloud Storage
---
It's pretty clear these days that {thing}-as-a-service is a pretty powerful concept. That said, it involves a certain level of trust in whoever is providing the service. Part of that is the trust that they'll keep providing you the service, but on the tin-foil-hat side, you trust that they won't use the information you give them to your detriment. This is especially true for storage/hosting service providers, where you're quite literally handing your data to a 3rd party.

I love Dropbox, but I'm not sure I'm ready to trust them with my secret plans for world domination. As such, I've decided that I'd like to add a layer of encryption on top of my Dropbox storage, as well as other similar providers. I looked around, and decided to work with [Dropbox](https://db.tt/xSBPWTGd), [Copy](https://copy.com?r=YubGHw), and [Google Drive](https://drive.google.com). As a disclaimer, the Dropbox and Copy links include my referral code so I get extra space if you use them.

<!--more-->

The first step was to set up these services to sync to a standardized location. I picked these 3 partially because all of them support auser-configurable sync location. For simplicity, I created ~/.cloud and set each provider to sync to ~/.cloud/{provider_name}, so I ended up with the following structure:

```
/Users/akerl/.cloud
├── Copy
├── Dropbox
└── Google
```

There was some content that I didn't plan on encrypting, so I set up symlinks linking those locations to where I wanted them in the filesystem:

``` bash
ln -s ~/.cloud/Dropbox ~/Dropbox # For convenient access as well as the various apps that sync via Dropbox
ln -s ~/.cloud/Google/Books ~/Books # I'm not terribly concerned about encrypting my collection of ebooks
```

I then looked at a couple encryption methods, and settled on [EncFS](https://en.wikipedia.org/wiki/EncFS). It has the benefit of operating at the file level and thus playing nice with the continuous syncing done by these providers, as well as working with FUSE to allow easy mounting on OSX or Linux. I'm only using the encrypted stores on my Macs right now, but the ability to easily expand is nice.

EncFS is packaged in [Homebrew](https://github.com/mxcl/homebrew), but the package depends on the [OSXFUSE](http://osxfuse.github.io/) packaged in Homebrew, which requires full XCode to install. I already had OSXFUSE installed via their package, and I wasn't particularly keen on installing XCode just to install OSXFUSE a second time. Thankfully, Homebrew makes it pretty easy to modify install "formulas", so I adjusted the EncFS formula to skip the OSXFUSE dependency and use the pre-existing libs. I threw that in a gist, and installing it was just a matter of pointing brew at it:

``` bash
brew install https://gist.github.com/akerl/7571256/raw/5799e52b8e69cd1e63f89426412678d5cb56f76d/encfs.rb
```

There was also an issue with the boost libraries being in the wrong spot, since I'm a rebel and don't put brew in /usr/local. I fixed that with the following one-liner:

``` bash
for x in /usr/local/brew/lib/libboost_*.dylib; do sudo ln -s $x /usr/local/lib; done
```

I then picked out the mount points I wanted and made the EncFS filesystems to put there. Setting up EncFS is pretty simple; you run the encfs command with a source and destination, and if they aren't set up, it walks you through setting them up. If they are, it prompts you for the key and mounts that filesystem. I ended up creating 3 mount points:

``` bash
encfs ~/.cloud/Dropbox/scratch.encfs ~/scratch
encfs ~/.cloud/Dropbox/tmp.encfs ~/tmp
encfs ~/.cloud/Copy/vault.encfs ~/vault
```

I ended up not setting up any encrypted filesystems on Google Drive after discovering an odd bug: their client blocks files from being "deleted" when they are deleted, so an EncFS mounted from Google Drive fails when trying to delete directories, claiming the aren't empty. It didn't look like something easily resolvable, and I had plenty of space from Dropbox and Copy to work with, so I resigned myself to using Google Drive for large but not sensitive files.

Part of the goal was to have these directories be mounted seamlessly, under the "If it's lots of work, I won't do it" principle. To do this, I added the credentials and path information to the OSX keychain. In Keychain Access, I added a password item for each encfs filesystem, and set the name to "EncFS", the account to the source path for the filesystem, and the password to the decryption key. I then edited the newly added item and set the comment to the destination path. I wrote the following script in Ruby that reads that information from the keychain and uses it to mount the filesystems:

(https://github.com/akerl/scripts/blob/master/mount_encfs)

``` ruby
#!/usr/bin/env ruby

require 'keychain'
require 'pathname'

encfs = ARGV.first || '/usr/local/brew/bin/encfs'

Keychain.generic_passwords.where(service: 'EncFS').all.each do |item|
  source = File.expand_path(item.account)
  dest = File.expand_path(item.comment)

  fail 'Paths are incorrect' unless Dir.exist? source and Dir.exist? dest
  next if Pathname.new(dest).mountpoint?

  mount = IO.popen "#{encfs} -S '#{source}' '#{dest}'", 'w'
  mount.puts item.password
  mount.close
end
```

Having run that manually and confirmed it worked by checking `df -h`, I set out to have the filesystems mounted automatically at boot. The easiest way to do to this on a Mac is via a launchd script, and [Lingon](http://www.peterborgapps.com/lingon/) provides a sweet interface for controlling launchd jobs. I added a job there pointing to the script, and rebooted to confirm it was happy:

``` bash
❯ df -h
Filesystem       Size   Used  Avail Capacity  iused    ifree %iused  Mounted on
/dev/disk1      233Gi   60Gi  173Gi    26% 15662492 45325027   26%   /
devfs           199Ki  199Ki    0Bi   100%      688        0  100%   /dev
map -hosts        0Bi    0Bi    0Bi   100%        0        0  100%   /net
map auto_home     0Bi    0Bi    0Bi   100%        0        0  100%   /home
encfs@osxfuse0  233Gi   60Gi  173Gi    26% 15662492 45325027   26%   /Users/akerl/vault
encfs@osxfuse1  233Gi   60Gi  173Gi    26% 15662492 45325027   26%   /Users/akerl/tmp
encfs@osxfuse2  233Gi   60Gi  173Gi    26% 15662492 45325027   26%   /Users/akerl/scratch
```

Success!

