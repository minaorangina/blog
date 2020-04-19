---
title: "How to copy a file from one git branch to another"
date: 2020-04-19T16:24:08+01:00
summary: Properly, without having to resort to copy/pasting (you're better than that).
draft: false
tags: ["git tricks"]
---

You're working on a branch. Let's say it's the `master` branch.

You have another branch. Let's call it `abandoned`.

You don't care about the `abandoned` branch, _except_ for one useful file. Let's call it `useful.txt`.

Don't do this:

{{< highlight bash "linenos=false" >}}
$ git checkout abandoned

[human manually copies `useful.txt` to their clipboard]

$ git checkout master

[human manually creates a new file and pastes into it]
{{< / highlight >}}
Instead, do this:
{{< highlight bash "linenos=false" >}}
$ git checkout master
$ git checkout abandoned useful.txt
{{< / highlight >}}
## 😸