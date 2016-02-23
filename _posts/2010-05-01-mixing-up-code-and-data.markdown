---
author: pcl
comments: true
date: 2010-05-01 08:12:56+00:00
layout: post
permalink: /2010/05/mixing-up-code-and-data/
slug: mixing-up-code-and-data
title: Mixing Up Code and Data
wordpressid: 305
categories:
- essays
---

From buffer overflows to cross-site scripting, decades of software security flaws can be traced back to a simple design problem: executable code (or otherwise specially meaningful data), and non-executable, black-box data are intermingled in the same channel. To execute arbitrary code, traditional buffer overflow exploits rely on non-executable data trampling execution state and eventually causing data to be executed as code. Cross-site scripting exploits, and all traditional injection exploits, work when intermediary systems fail to identify the difference between code and data in exactly the same way as some other system.

To prevent these exploits, developers are generally advised to canonicalize input and encode data in output. While this is correct, I think it is important to also understand that _it shouldn't be that way_. Developers should have to go an extra mile to cause data to be interpreted as code, not the other way around.
<!-- more -->
Consider this common SQL injection scenario:

~~~ php
$result = mysql_query( "select id, name, pass from users where name='$name'", $db );
~~~

While it's clear that $name is simply data to the developer, it's being crammed into a string along with code and handed off to another system for re-interpretation. A common solution to prevent injection is something like this:

~~~ php
$query = sprintf( "select id, name, pass from users where name='%s' ", mysql_real_escape_string($name) );
$result = mysql_query( $query, $db );
~~~

It's true that this solution fixes the vulnerability in this instance, but it's a heavy burden: always remember to add a whole bunch of code every time you perform this incredibly common task, or you will introduce one of the most serious vulnerabilities possible into your application. In my opinion, even though the code may be "secure," it is still wrong. You are still mixing up code and data in the same place on the same channel. Here is a better solution:

~~~ php
$stmt = $mysqli->prepare("select id, name, pass from users where name=?");
$stmt->bind_param("s", $name);
$stmt->execute();
~~~

This looks similar, but it's important to understand how it works. The first line sends all of the code to be parsed and prepared by the server. Data to be filled in later is represented by question marks, which can only be used where data is expected. After this point, nothing is re-interpreted. When we send the data on the second line, the server is only reading data, and no special encoding is required to prevent it from being treated as code.

Even though it suffers the same criticism of being extra code every time you want to execute a SQL query, a developer used to coding this way is much less likely to make a mistake than a developer used to the previous example. The statements could also be prepared in an initializer somewhere completely different instead of right next to where data is being bound, to make it even harder to make a mistake.

Imagine if HTML/HTTP had something similar:
[html light="1"]
<html>
<div class="title">&data[0,16];</div>
<div class="body">&data[17,256];</div>
</html>
Data until EOF. <!-- &amp; No possibility of being interpreted as html. <script>alert(1);</script>
[/html]

Note that the data part is a complete black box and terminated out-of-band (by closing the connection). There aren't lengths or other special markers embedded in the data. I'm not suggesting this is a realistic possibility for HTML or any other established standard. I just want to make the point that a protocol which makes it impossible to confuse code and data is invulnerable to injection attacks by default.

A related idea is identifying the source of data. When an application includes user input in its own output, or in SQL statements, or anywhere else, it takes responsibility for it; to other applications, it's data (or code) coming from the first application, not data coming from a user or third-party application somewhere else.

In cases where user input needs to be specially interpreted, especially by some other application - when you want to let people include a little HTML in a post, for example - then you have a more difficult problem which can't be solved by shoving it all into a separate black box. But imagine how nice it would be if you could specify that some bit of HTML was from a user, or some other website (perhaps with digital signatures), or whatever, and the browser could handle sandboxing and cross-domain policies and everything else it already has the ability to do.

I think it is important to address this idea up front not just when designing a protocol or file format, but when designing APIs and internal interfaces to existing systems. Even if you have to output HTML or actual SQL statements, an interface to generate the output which cleanly separates code and data can relieve the burden from developers, isolate potential weaknesses in one place, and go a long way to improve the security of a system.
