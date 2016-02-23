---
id: 253
title: 'TIP: Make bash tab completion ignore .svn directories'
date: 2010-03-16T11:50:45+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=253
permalink: /2010/03/tip-make-bash-tab-completion-ignore-svn-directories/
categories:
  - tips
tags:
  - bash
  - tips
---
Having to tab through the fifty million otherwise empty &#8220;net/mycompany/project/unit/subunit&#8221; directories that the Java ecosystem necessitates has consistently driven me crazy because completion stops at each step to let me choose the .svn directory, and I have to look and type the first letter of the directory I actually want to make it continue.

It&#8217;s actually really easy to fix this:

<pre class="brush: bash; light: true; title: ; notranslate" title="">export FIGNORE=.svn</pre>

<tt>$FIGNORE</tt> is just a colon-separated list of suffixes to ignore when doing tab completion.