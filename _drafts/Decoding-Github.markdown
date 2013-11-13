---
layout: post
title: Decoding GitHub's Contribution Charts
---
I've been attempting the commit challenge, or the GitHub challenge, or whatever the "official" term is for "I'm going to commit every day." It's been going pretty good, and it's certainly pushed me to be productive when I'd have otherwise slacked off. As part of this, I became a pretty big fan of GitHub's contribution chart:

![Official GitHub Chart](/assets/images/github_chart.png)

Their chart provides a solid visual indication of both how often you commit and how your intensity fluctuates. I didn't know exactly what I'd do with my own copy of the chart, but I decided I wanted one. Since scraping their SVG seemed rather untidy, that meant I needed to make my own.

# What's out there?

There are a few existing projects that deal with charts modeled after GitHub's. A bunch of them, like [Contributions.io](http://contribution.io/), are concerned with encoding text or other symbols into the chart. Others, like [git-cal](https://github.com/k4rthik/git-cal), let you use other data (like a local git repo) as a source.

These tools are all cool, but I wanted to have both the distinctive look of the GitHub chart along with the valid data.

