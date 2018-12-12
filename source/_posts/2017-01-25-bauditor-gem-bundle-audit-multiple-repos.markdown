---
layout: post
title: "Bauditor gem - bundle-audit multiple repos"
date: 2017-01-25 17:18:03 -0600
comments: true
categories: [ruby]
keywords: ruby,rails,bundler, bundler-audit, security
---

I was adding [bundler-audit](https://github.com/rubysec/bundler-audit) to a ruby application recently and I had this idea that it would be really nice to be able to run it on multiple repositories at once. It's great when you get the status as part of a CI run or a rake task, but if you have some applications that are seldom updated it's important to audit them as well. So I made a thing.

Introducing [bauditor](https://github.com/leklund/bauditor), a simple gem to help you run bundle-audit on multiple repositories in one pass. It will take a config file with a list of git repositories or you can pass them in on the command line:

```
$ bauditor help audit
Usage:
  bauditor audit

Options:
  r, [--repos=one two three]
  c, [--config=CONFIG]

run bundle-audit on multiple repositories
```

It will clone each repo into a tempdir and then run `bundle-audit`. It prints a handy summary at the end. It cleans up after each run so it's not super fast yet. One of the features I want to add is the ability to persist the repositories as well as specify the repository path.

Check it out on rubygems: [bauditor 0.2.2](https://rubygems.org/gems/bauditor). Or get the source on [GitHub](https://github.com/leklund/bauditor). 
