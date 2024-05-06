---
date: "2016-04-27T00:00:00Z"
title: 'Code and Taxes: Totally Exciting'
description: Using ledger and simple scripts to automate double entry accounting
aliases:
- /post/2016-04-27-code-and-taxes-totally-exciting
tags:
- code
- finances
---

Now that tax day has officially passed, it occurred to me that the best way to celebrate would be to plan for next year. A few of my friends semi-seriously keep tabs on a lightweight challenge: try to break as close to zero on tax day. I don't know how much actual effort most of them put into this challenge, but I'd average it's low, because I've not historically put effort into it. But the goal is fairly sound: don't give Uncle Sam an interest free loan of your hard-earned cash.

It's also dawned on me that, complex though tax code may be, I have a computer and a marginal understanding of math. As such, I've set out to code my way to victory.

<!--more-->

Where I Was
===========

Previously, I mostly only looked *forward* financially. I did this primarily with [this ruby script](https://github.com/akerl/scripts/blob/master/old/budget), whose strategy was simple: model all my monthly bills as YAML, and then roll forward into the future and model the impact. An example YAML file:

{{< gist akerl 1204df254270d4e76a65965f926f7223 >}}

And example output:

{{< gist akerl 700d25e9370fca4b6cfe31e880cfd90c >}}

This kind of output was amazingly useful for my use case at the time: it's lightweight, it was easy to set up, and for seeing "how does it affect my next 6 months if I crank up my 401k contributions", it did spendidly.

Unfortunately, it has some shortcomings:

* It operates with a single "pool"; there's no accounting for multiple savings accounts
* Because of that, it can't model things like "put money into savings", which costs my checking account money but gives my savings account money
* Most importantly for our purposes: it doesn't remember the past

The last point is critical for my "Winning The Tax Game" crusade: taxes are a year-long tug-of-war, so I needed not just a projection from now til Christmas, I also needed to look *back* at how money had moved up to this point.

A Change Of Scenery
============

I did some research and it turns out that while I was hand-rolling Ruby scripts to do budget projections, other (saner) folks were using existing open source tools with far more standardization and bells and whistles. In the category of command-line accounting tools, [ledger](http://www.ledger-cli.org/) and its many forks is very popular. It's possible I'll do a longer post in the future on how awesome ledger is, but for now these are the major factors:

1. It does [double entry accounting](https://en.wikipedia.org/wiki/Double-entry_bookkeeping_system), which is a fancy way of saying it very explicitly tracked which accounts money goes into and out of
2. It supports transactions in the future
3. The [transaction format](http://www.ledger-cli.org/3.0/doc/ledger3.html#Basic-format) it uses is easily parsable in code (at least, the basic subset is, which is all I needed): 

The first point means I can get the per-account tracking that I was lacking. This is super important because I need to say things like "from my paycheck, this much money went to each kind of tax withholding":

{{< gist akerl 9c5ca3cd7c48ecc1ffd27078ab5b43b4 >}}

The second means I can do projections using ledger, by adding in future transactions, allowing it to replace my budget Ruby script.

The 3rd means I can do so with code, by programmatically generating those future transactions.

The Part You've Been Waiting For
=======================

Step 1 was to port my budget script so I could do ledger-format projections. Because I love composable libraries, I started by writing a lightweight tool that reads and writes basic ledger journals: [libledger](https://github.com/akerl/libledger)

This tool is used by my [ballista](https://github.com/akerl/ballista) gem, named for a seige weapon that projects missiles across the battlefield. Ballista consumes a YAML file that looks very similar to my original budget script's config, except with ledger-format account names:

{{< gist akerl 81046cc748b6f7478744359e2e7d8c89 >}}

It uses these to generate libledger objects and then dump those out as journal files:

{{< gist akerl 796950fdb061f7805b592c9de9eef896 >}}

This gets me most of the way, but now I need to link them up to historical data. I settled on a fairly arbitrary layout for my ledger journals: directories per year, with files per month:

{{< gist akerl 7352a4aa3f2f6d8dcd115b9d503d9012 >}}

I put the opening balance info in last year (so that it wouldn't get confused with this year for tax calculations) and filled out my pay info for this year so far. Then I wrote up [a quick Ruby script](https://github.com/akerl/ledgerhelpers/blob/master/ldgproject) that uses Ballista to generate per-month projections and dump them into the journal dir. Notably, I taught it to [mark its entries with a comment](https://github.com/akerl/ledgerhelpers/blob/b69b01d2cb2d6fcabacb560af8e8bc371b3dc6d4/ldgproject#L32), so it wouldn't clobber anything I'd already added to this month.

So now I've got journals for the past and future. I took a break from coding to relax and read lots of fun tax documentation. I then distilled the relevant parts into YAML:

{{< gist akerl 5250ff5c6481eb1eed64901688f9ec05 >}}

From there, I wrote [taxcalc](https://github.com/akerl/ledgerhelpers/blob/master/taxcalc). It uses the taxes.yml and does a printout of tax liability, first per-tax and then summed:

{{< gist akerl e21df18985a2689766aa5a10b28d047c >}}

To note, it *does* assume that `ledger` without other args will know which journals you want to talk to. The easiest way to accomplish that is by setting ~/.ledgerrc to contain "--file /path/to/my/core.ldg", or set LEDGER_FILE in the environment.

So now we're there! It's now easy to tweak the projection and re-run ldgproject, which will update the projection for the year, and then run taxcalc to show how that impacts your tax situation.

Next steps
==========

More flexible projections
-------------

Right now, ballista is limited by some of the same boundaries as my original budget script: It works great for monthly actions, but it's hard (read: prohibitively annoying) to model other cadences. Specifically, I have an Amazon Subscribe And Save order that bills every 4 months, which I'd like to model as a known recurring expense. I expect I'll upgrade the "when:" logic to be more flexible (maybe with another library!). I may take some inspiration from ledger's own [period expressions](http://www.ledger-cli.org/3.0/doc/ledger3.html#Period-Expressions), which are amazingly flexible.

Support for more ledger journal layouts
--------

The ldgproject script is hardcoded with my expectation of the journal directory structure, but there's no real reason it needs to stay that way. I'll likely do some tweaking with formatted strings to allow arbitrary journal directories.

Winning the tax game
-------------

If I don't win this year, next year I'll be blogging about building a tax robot.

Smoother entry code for ledger itself
---------

My primary day-to-day expenses occur in a flurry of sub-$20 transactions. I'm trying to find a balance between drowing in ledger entries and not getting signal on my spending, so I'm considering setting up some kind of twilio script or similar that would let me easily provide the minimal info and have it extrapolate ledger entries.

