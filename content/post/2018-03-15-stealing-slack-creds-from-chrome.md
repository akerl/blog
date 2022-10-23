---
date: "2018-03-15T00:00:00Z"
title: Stealing Slack Creds from Chrome
description: Using Chrome cookies to extract Slack API credentials
aliases:
- /post/2018-03-15-stealing-slack-creds-from-chrome
---

A while back, I wanted to do a couple quick things w/ the Slack API. The script I was writing would only end up being run a handful of times, all from my local computer, and I hate having multiple distinct credentials stored in the same place with the same perms, so I hatched a plan: piggyback on the existing creds my browser was using to access Slack.

<!--more-->

As a disclaimer: this issue isn't Chrome-specific, I just happen to use Chrome to access Slack. I'm pretty confident this kind of credential access could be done with any other browser, and potentially also with the "native" Slack app.

Time for exploring
==================

I led my investigation with the understanding that when I authenticated to Slack, their server was providing a token to Chrome, which Chrome could then use for subsequent requests to prove my identity. So I headed to the directory where Chrome stores its cache of cookies, localstorage, and other data: `~/Library/Application Support/Google/Chrome`.

I also had another bit of information: I knew that Slack tokens (at least the ones I'd run into before) had a static prefix of "xoxs-". My first grep looked promising:

{{< gist akerl 873601c42cad0868c23dc15817985b9b >}}

Reading LocalStorage
====================

As the file paths suggest, Chrome stores LocalStorage in [LevelDB](https://github.com/google/leveldb), a key-value store. That the on-disk format is binary rather than plaintext is annoying, but not as much as the other limitation: LevelDB is designed to be accessed by only one process at a time. Since Chrome is using the database, all the libraries I checked weren't able to open up the database concurrently, even just for reads. Either they refused due to the database being in use, or else they'd load it but not see an up-to-date picture of the data.

On a whim, I tried a more blunt approach: I opened one of the log files in `less`, proceeded through its warning that the file including non-text data, and searched for "xoxs-", our known Slack token prefix. Sure enough, I found matches, surrounded by binary garbage.

I decided I didn't really care for elegant parsing, I just needed to pull the raw tokens out. The token is stored in LocalStorage along with info on the slack subdomain (and thus Slack team) that it's for, but I didn't even need that, since once I had the token extracted, I could use it to ask the Slack API what team it was for.

I opened up my Ruby REPL and wrote a quick one-liner with a regex pattern to find Slack tokens. Forcing the encoding let Ruby discard the non-text data and run my regex pattern across the rest: `File.read(db).force_encoding('ASCII-8BIT').scan(/xoxs-\d+-\d+-\d+-\h+/)`

Cleaning it all up
=========

So now I knew how to get the tokens out. My next step was to make a quick module for that, [which I've released here.](https://github.com/akerl/limp)

The library provides a super simple `Limp.tokens` method that returns an array of Slack tokens it's found. There's also a `limp` command which returns the tokens on the command line:

{{< gist akerl 8149c93b585665b549be6addcbd456ae >}}

