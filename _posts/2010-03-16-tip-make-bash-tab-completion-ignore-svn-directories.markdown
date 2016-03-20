---
author: pcl
comments: true
date: 2010-03-16 15:50:45+00:00
layout: post
permalink: /2010/03/tip-make-bash-tab-completion-ignore-svn-directories/
slug: tip-make-bash-tab-completion-ignore-svn-directories
title: 'TIP: Make bash tab completion ignore .svn directories'
wordpressid: 253
categories:
- tips
tags:
- bash
- tips
---

Having to tab through the fifty million otherwise empty "net/mycompany/project/unit/subunit" directories that the Java ecosystem necessitates has consistently driven me crazy because completion stops at each step to let me choose the .svn directory, and I have to look and type the first letter of the directory I actually want to make it continue.

It's actually really easy to fix this:

~~~ bash
export FIGNORE=.svn
~~~

$FIGNORE is just a colon-separated list of suffixes to ignore when doing tab completion.
