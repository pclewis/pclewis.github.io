---
id: 364
title: 'TIP: Fixing author in git history'
date: 2010-10-23T18:34:36+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=364
permalink: /2010/10/tip-fixing-author-in-git-history/
categories:
  - tips
---
I always forget to set up my user info on git on new machines before I check stuff in. It&#8217;s pretty easy to fix if there&#8217;s nobody else in your repo:

<pre class="brush: bash; title: ; notranslate" title="">git filter-branch --env-filter "\
  export GIT_AUTHOR_NAME=Dade\ Murphy \
         GIT_AUTHOR_EMAIL=zer0cool@example.com \
         GIT_COMMITTER_NAME=Dade\ Murphy \
         GIT_COMMITTER_EMAIL=zer0cool@example.com "
</pre>

Source: [serverfault](http://serverfault.com/questions/12373/how-do-i-edit-gits-history-to-correct-an-incorrect-email-address-name)