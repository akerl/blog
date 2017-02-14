---
layout: post
title: Reverse engineering GitHub streaks
---

To say I'm addicted to GitHub is an understatement. But I've attempted to focus my addiction towards productive goals, and so I decided that I wanted to process GitHub streak data programmatically. To my dismay, streak data isn't exposed as part of their API, and my request that they add it was met with polite neutrality. So I set out to see how their site built the streak chart on [the user page](https://github.com/akerl).

When I began this adventure, I knew enough JavaScript to shoot myself in the foot and I'd never dealt with large existing JavaScript codebases, but I nonetheless dove in to Chrome's Developer Tools to dissect how the page created that chart. I got my first win from [Jon Chen](https://github.com/bsdlp), who identified the source of the data: a JSON array of dates and scores served at https://github.com/users/{username}/contributions_calendar_data (a URL that no longer works, which we'll get to). This gave me the raw score data, and I got to work building a module around that.

<!--more-->

githubstats
===========

I'm a firm believer that small, focused libraries are the best foundation for solid code. In keeping with that, I first set to work building a module that consumed the raw streak data and exposed it with some helper methods. The result is [githubstats](https://github.com/akerl/githubstats). It was one of my first Ruby projects, and it turned out to be a pretty solid intro to Ruby programming.

I quickly realized that I was dealing with 2 different "objects", GitHub users and the underlying stats bundles, and architected my module accordingly. Unfortunately, this made using the module quite obnoxious: you'd have to create a GithubStats::User object, and accessing the data would require user.data.{method} for every call. The first thing I did to simplify this was adding a helper method to the module itself:

{% gist akerl/a0c207added29ee4761e %}

This ended up being a pattern I copied for most of my future modules: the module has a class method .new() that is proxied to the most commonly desired class.

To solve the user.data.{method} problem, I decided to do some more complicated proxying. Thankfully, I'd recently completed the [Ruby Koans](http://rubykoans.com/) (which I highly recommend), and one of the sections is about method_missing and how Ruby handles method calls. I decided to override method_missing and respond_to? on the User object, so that any method called on User that didn't exist but *did* exist on .data would be proxied through. Initially, I had it proxy the calls directly from method_missing, but I read that the process of checking method_missing was fairly intensive, compared to making direct method calls, so I settled on a solution:

{% gist akerl/4faa309b1ee727037255 %}

This is, in theory, better for performance at the cost of some code cleanliness. For example, the first time user.mean is called, method_missing is invoked, and it confirms that user.data will respond to .mean. It then dynamically adds a method to the user object for user.mean, which calls @data.mean, and then calls that method. Future calls to user.mean will just use the newly created method directly, rather than hitting method_missing. At the advice of the koans, I also overrode respond_to?(), so that it accurately reflects what calls will succeed.

Having completed an initial version of the data library, I set out to find an actual use for it.

githubchart
===========

In an interesting turn of events, I decided that it would be awesome to have my own SVG copy of the GitHub contribution streak chart. At the time, I figured it would be cool to display on my blog or elsewhere. With this in mind, I started work on [githubchart](https://github.com/akerl/githubchart).

Going into this project, I expected to spend some time learning how to draw SVGs in Ruby, a decent amount of time figuring out how to handle command line utilities, and about 3 seconds checking GitHub to figure out where the breakpoints between the various colors fell. My estimates were shockingly terrible.

Drawing SVGs in Ruby is pretty easy. I ended up using [rasem](https://github.com/aseldawy/rasem), which is a pretty solid library, despite missin support for some fun things like title attributes that I wish it had. The command line tool was similarly easy, I dropped OptionParser in, created a GithubChart::SVG object, and dropped the result into a file. The massive timesink turned out to be deciphering GitHub's color picking logic.

It turns out that they don't use static break-points between color levels. That part was immediately apparent. After realizing that, I grabbed a bunch of peoples' data and tried brute-forcing it, expecting to find some linear algorithm or other obvious pattern. That was a total failure. From there, I decided to dig in to their actual Javascript. It was minified to the point of insanity, but over the course of several weeks I broke it apart and commented it out:

{% gist akerl/47b614a1503a2f6f7992 %}

The first realization was that they're doing some outlier math. I later learned that this is two-pass variance, but at the time I just knew it looked weird and set to work replicating it:

{% gist akerl/bf8d2345df466284fd92 %}

You'll note that they do some weird things with outlier counts. They bump a set number of outliers, and any numbers that match those outliers. So if your outliers are, in order, "20, 30, 40, 50, 100, 20", then any instances of 20, 30, and 40 are rejected from the set, leaving 50 and 100 behind. That tripped my up for quite a while until I really dug in to what the JS was doing.

The next component was getting the layout of the SVG right. Doing the grid squares was pretty easy, but it turns out there's some cute math involved in the month labels to decide when they show up / where they show up. This all happens in the return statement, which happens right at the pottom of the last gist:

{% gist akerl/43fa29c1cac775b65838 %}

The very important thing to note, which threw me off for a long time, is that they use $week_number. This skews results for January, because January 1st is always the start of a new week_number, even if it falls mid-week. They count how many week_numbers start in each month and give each month label that many columns of space. They also check the front/end and hide the first/last label if they're going to overlap other labels.

I spent a lot of time just tweaking pixel counts, and the end result was pixel-perfect matching to GitHub's graphs. For now. I seem to keep finding things that I need to tweak.

Next steps
==========

I've already made some major tweaks to the code, some based on feedback and some based on whimsy.

Better pipeline support
-------------------

I've made some changes and merged some PRs in response to [Stan's](http://www.schwertly.com/2014/09/creating-github-style-contribution-graphs-for-anything/) usage, which involves pumping self-generated data in to GithubChart. To support this, he submitted [a pull request](https://github.com/akerl/githubchart/pull/31) to support providing score data on STDIN.

He also is a big fan of piping together commands, so I [added support](https://github.com/akerl/githubchart/commit/d99e6a465e445786a232607e4ae87b250ee81012) for printing the SVG to STDOUT rather than to a file.

Alternate colors
----------------

I baked in support for changing the colors for the chart from the start, but didn't expose that to the script. Then on Halloween, GitHub switched up their color scheme to more festive colors, and I of course needed to follow suit. The [resulting change](https://github.com/akerl/githubchart/commit/7c90bcc9b07965cb978a11a29981a03433aa7ab2) turned out to be pretty easy: I grabbed the new colors, added an option to the script, and tweaked the defaults to support named color schemes.

Actual usage?
-------------

The one thing I've not yet done with the charts? Actually use them. I have used the data in a couple fun projects, like [this twilio endpoint](https://github.com/akerl/committed) that will tell me if I've committed today.

