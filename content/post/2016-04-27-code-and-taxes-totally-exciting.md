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

```yaml
---
- name: payday
  type: income
  when:
    - 15
    - 30
  amount: 2730
- name: rent
  when: 2
  amount: 2760
- name: electric
  when: 4
  amount: 40
- name: internet
  when: 10
  amount: 80
- name: github
  when: 12
  amount: 7
- name: geico
  when: 5
  amount: 64
- name: spotify
  when: 10
  amount: 10
- name: tmobile
  when: 2
  amount: 35
- name: netflix
  when: 30
  amount: 12
- name: savings
  when: 2
  amount: 1000
- name: incidentals
  when: 2
  amount: 1000
```

And example output:

```
❯ budget finances.yml 5000 2 # This will project out 2 months, with an assumed starting balance of 5000
2016-04-28 |       5000 |          0 | initial value
2016-04-30 |       7730 |       2730 | payday
2016-04-30 |       7718 |        -12 | netflix
2016-05-02 |       4958 |      -2760 | rent
2016-05-02 |       4923 |        -35 | tmobile
2016-05-02 |       3923 |      -1000 | savings
2016-05-02 |       2923 |      -1000 | incidentals
2016-05-04 |       2883 |        -40 | electric
2016-05-05 |       2819 |        -64 | geico
2016-05-10 |       2739 |        -80 | internet
2016-05-10 |       2729 |        -10 | spotify
2016-05-12 |       2722 |         -7 | github
2016-05-15 |       5452 |       2730 | payday
2016-05-30 |       8182 |       2730 | payday
2016-05-30 |       8170 |        -12 | netflix
2016-06-02 |       5410 |      -2760 | rent
2016-06-02 |       5375 |        -35 | tmobile
2016-06-02 |       4375 |      -1000 | savings
2016-06-02 |       3375 |      -1000 | incidentals
2016-06-04 |       3335 |        -40 | electric
2016-06-05 |       3271 |        -64 | geico
2016-06-10 |       3191 |        -80 | internet
2016-06-10 |       3181 |        -10 | spotify
2016-06-12 |       3174 |         -7 | github
2016-06-15 |       5904 |       2730 | payday
```

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

```
2016/04/15 *  Pay Check
    Expenses:Taxes:federal_income                      $729.26
    Expenses:Taxes:va_income                           $215.14
    Expenses:Taxes:social_security                     $299.46
    Expenses:Taxes:medicare                             $70.04
    Assets:Checking:simple                            $2739.10
    Assets:401K:Trad:work                              $772.00
    Income:Salary:work                               $-4825.00
```

The second means I can do projections using ledger, by adding in future transactions, allowing it to replace my budget Ruby script.

The 3rd means I can do so with code, by programmatically generating those future transactions.

The Part You've Been Waiting For
=======================

Step 1 was to port my budget script so I could do ledger-format projections. Because I love composable libraries, I started by writing a lightweight tool that reads and writes basic ledger journals: [libledger](https://github.com/akerl/libledger)

This tool is used by my [ballista](https://github.com/akerl/ballista) gem, named for a seige weapon that projects missiles across the battlefield. Ballista consumes a YAML file that looks very similar to my original budget script's config, except with ledger-format account names:

```yaml
- name: Pay Check
  when:
  - 15
  - 30
  actions:
    Expenses:Taxes:federal_income: $729.26
    Expenses:Taxes:va_income: $215.14
    Expenses:Taxes:social_security: $299.46
    Expenses:Taxes:medicare: $70.04
    Assets:Checking:simple: $2739.10
    Assets:401K:Trad:work: $772.00
    Income:Salary:work: $-4825.00
- name: Automatic transfer to Savings
  when: 2
  actions:
    Assets:Savings:ally: $1000.00
    Assets:Checking:simple: null
- name: Rent
  when: 2
  actions:
    Expenses:Bills:rent: $2800.00
    Assets:Checking:simple: null
- name: Electric Bill
  when: 4
  actions:
    Expenses:Bills:electric: $40.00
    Assets:Checking:simple: null
- name: Comcast Bill
  when: 10
  actions:
    Expenses:Bills:internet: $82.95
    Assets:Checking:simple: null
- name: Github Bill
  when: 14
  actions:
    Expenses:Bills:github: $7.00
    Assets:Checking:simple: null
- name: Spotify
  when: 10
  actions:
    Expenses:Bills:spotify: $9.99
    Assets:Checking:simple: null
- name: Geico
  when: 5
  actions:
    Expenses:Insurance:auto: $65.73
    Assets:Checking:simple: null
- name: T-Mobile
  when: 2
  actions:
    Expenses:Bills:phone: $30.50
    Assets:Checking:simple: null
- name: Netflix
  when: 29
  actions:
    Expenses:Bills:netflix: $11.99
    Assets:Checking:simple: null
```

It uses these to generate libledger objects and then dump those out as journal files:

```
❯ ballista config.yml -m1
2016/04/29 ! Netflix
    Expenses:Bills:netflix                              $11.99
    Assets:Checking:simple

2016/04/30 ! Pay Check
    Expenses:Taxes:federal_income                      $729.26
    Expenses:Taxes:va_income                           $215.14
    Expenses:Taxes:social_security                     $299.46
    Expenses:Taxes:medicare                             $70.04
    Assets:Checking:simple                            $2739.10
    Assets:401K:Trad:work                              $772.00
    Income:Salary:work                               $-4825.00

2016/05/02 ! Automatic transfer to Savings
    Assets:Savings:ally                               $1000.00
    Assets:Checking:simple

2016/05/02 ! Rent
    Expenses:Bills:rent                               $2800.00
    Assets:Checking:simple

2016/05/02 ! T-Mobile
    Expenses:Bills:phone                                $30.50
    Assets:Checking:simple

2016/05/04 ! Electric Bill
    Expenses:Bills:electric                             $40.00
    Assets:Checking:simple

2016/05/05 ! Geico
    Expenses:Insurance:auto                             $65.73
    Assets:Checking:simple

2016/05/10 ! Comcast Bill
    Expenses:Bills:internet                             $82.95
    Assets:Checking:simple

2016/05/10 ! Spotify
    Expenses:Bills:spotify                               $9.99
    Assets:Checking:simple

2016/05/14 ! Github Bill
    Expenses:Bills:github                                $7.00
    Assets:Checking:simple

2016/05/15 ! Pay Check
    Expenses:Taxes:federal_income                      $729.26
    Expenses:Taxes:va_income                           $215.14
    Expenses:Taxes:social_security                     $299.46
    Expenses:Taxes:medicare                             $70.04
    Assets:Checking:simple                            $2739.10
    Assets:401K:Trad:work                              $772.00
    Income:Salary:work                               $-4825.00
```

This gets me most of the way, but now I need to link them up to historical data. I settled on a fairly arbitrary layout for my ledger journals: directories per year, with files per month:

```
❯ tree
.
├── core.ldg
├── journals
│   ├── 2015
│   │   └── 12.ldg
│   └── 2016
│       ├── 01.ldg
│       ├── 02.ldg
│       ├── 03.ldg
│       └── 04.ldg
└── projections.yml

❯ cat core.ldg
include ./journals/2015/*.ldg
include ./journals/2016/*.ldg
```

I put the opening balance info in last year (so that it wouldn't get confused with this year for tax calculations) and filled out my pay info for this year so far. Then I wrote up [a quick Ruby script](https://github.com/akerl/ledgerhelpers/blob/master/ldgproject) that uses Ballista to generate per-month projections and dump them into the journal dir. Notably, I taught it to [mark its entries with a comment](https://github.com/akerl/ledgerhelpers/blob/b69b01d2cb2d6fcabacb560af8e8bc371b3dc6d4/ldgproject#L32), so it wouldn't clobber anything I'd already added to this month.

So now I've got journals for the past and future. I took a break from coding to relax and read lots of fun tax documentation. I then distilled the relevant parts into YAML:

```yaml
taxable: # This uses the same account names from my Ledger journal and lists all my taxable income
- Income:Salary:work
- Income:Bonus:work
deductions: # These are things that can be deducted
- name: Expenses:Commute:metro # If there's no amount given, the script looks it up as a ledger account and takes the annual total
- name: Assets:401K:Trad:work
  from: # If it only applies for some taxes, call out which ones
  - federal_income
  - va_income
- name: Standard Federal Deduction
  amount: 6300 # If it's a static amount, just call out how much
  from: federal_income
- name: Standard VA Deduction
  amount: 3000
  from: va_income
- name: Personal Federal Exemption
  amount: 4050
  from: federal_income
- name: Personal VA Exemption
  amount: 930
  from: va_income
brackets: # Each key here is the name of a tax. The names match with the ledger journal's entries for the *withholdings* for that tax
  federal_income: # Each tax has an array of its brackets
  - rate: .10
    starts: 0 # Call out the bottom of that bracket
  - rate: .15
    starts: 9276
  - rate: .25
    starts: 37651
  - rate: .28
    starts: 91151
  - rate: .33
    starts: 190151
  - rate: .35
    starts: 413351
  - rate: .396
    starts: 415051
  va_income:
  - rate: .02
    starts: 0
  - rate: .03
    starts: 3001
  - rate: .05
    starts: 5001
  - rate: .0575
    starts: 17001
  medicare:
  - rate: .0145 # Think how much simpler this would be if all taxes were flat?
    starts: 0
  social_security:
  - rate: .062
    starts: 0
```

From there, I wrote [taxcalc](https://github.com/akerl/ledgerhelpers/blob/master/taxcalc). It uses the taxes.yml and does a printout of tax liability, first per-tax and then summed:

```
❯ ./scripts/taxcalc taxes.yml
Taxes:
  federal_income
    Deductions:
      Expenses:Commute:metro:  ------------  1300.00
      Assets:401K:Trad:work:  ------------  18648.82
      Standard Federal Deduction:  --------  6300.00
      Personal Federal Exemption:  --------  4050.00
    Taxable Income:  --------------------  110167.86
    Taxes Per Bracket:
      0.28:  ------------------------------  5325.00
      0.25:  -----------------------------  13375.00
      0.15:  ------------------------------  4256.25
      0.1:  --------------------------------  927.60
    Total Assessed:  ---------------------  23883.85
    Total Withheld:  ---------------------  23321.66
    Total Owed:  ---------------------------  562.19
  va_income
    Deductions:
      Expenses:Commute:metro:  ------------  1300.00
      Assets:401K:Trad:work:  ------------  18648.82
      Standard VA Deduction:  -------------  3000.00
      Personal VA Exemption:  --------------  930.00
    Taxable Income:  --------------------  116587.86
    Taxes Per Bracket:
      0.0575:  ----------------------------  5726.30
      0.05:  -------------------------------  600.00
      0.03:  --------------------------------  60.00
      0.02:  --------------------------------  60.02
    Total Assessed:  ----------------------  6446.32
    Total Withheld:  ----------------------  6500.05
    Total Owed:  ---------------------------  -53.73
  medicare
    Deductions:
      Expenses:Commute:metro:  ------------  1300.00
    Taxable Income:  --------------------  139166.68
    Taxes Per Bracket:
      0.0145:  ----------------------------  2017.93
    Total Assessed:  ----------------------  2017.93
    Total Withheld:  ----------------------  2019.67
    Total Owed:  ----------------------------  -1.74
  social_security
    Deductions:
      Expenses:Commute:metro:  ------------  1300.00
    Taxable Income:  --------------------  139166.68
    Taxes Per Bracket:
      0.062:  -----------------------------  8628.40
    Total Assessed:  ----------------------  8628.40
    Total Withheld:  ----------------------  8635.65
    Total Owed:  ----------------------------  -7.25

Total Income:  --------------------------  140466.68
Total Assessed:  -------------------------  40976.50
Total Withheld:  -------------------------  40477.03
Total Owed:  -------------------------------  499.47
```

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

