---
author: pcl
comments: true
date: 2010-10-23 22:34:36+00:00
layout: post
permalink: /2010/10/tip-fixing-author-in-git-history/
slug: tip-fixing-author-in-git-history
title: 'TIP: Fixing author in git history'
wordpressid: 364
categories:
- tips
---

I always forget to set up my user info on git on new machines before I check stuff in. It's pretty easy to fix if there's nobody else in your repo:

~~~ bash
git filter-branch --env-filter "\
  export GIT_AUTHOR_NAME=Dade\ Murphy \
         GIT_AUTHOR_EMAIL=zer0cool@example.com \
         GIT_COMMITTER_NAME=Dade\ Murphy \
         GIT_COMMITTER_EMAIL=zer0cool@example.com "
~~~

Source: [serverfault](http://serverfault.com/questions/12373/how-do-i-edit-gits-history-to-correct-an-incorrect-email-address-name)

