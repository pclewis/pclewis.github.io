---
author: pcl
comments: true
date: 2010-03-26 20:13:54+00:00
layout: post
permalink: /2010/03/tip-using-gdb-as-an-interactive-c-shell/
slug: tip-using-gdb-as-an-interactive-c-shell
title: 'TIP: Using GDB as an Interactive C Shell'
wordpressid: 70
categories:
- tips
tags:
- c/c++
- gdb
- tips
---

Many programming languages come with some way to run an interactive shell, or [REPL (read-eval-print loop)](http://en.wikipedia.org/wiki/REPL). This makes it extremely easy to test little bits of code and understand exactly what they do, and is invaluable when learning a new language or library. For example:

What's the result of `(unsigned int)atoi("4294967295")` in C?

Even if you know the answer, how quickly can you prove it? How concisely can you communicate the proof via IM or email? What if it's a poorly documented third-party library function, and not a standard one?

For quick tasks, you can just use [gdb](http://www.gnu.org/software/gdb/) which is probably already present on any system that has [gcc](http://gcc.gnu.org/). Just fire up gdb on any binary, set a breakpoint on main, and run. When it stops you will be able to call functions and examine their results, and many other common REPL tasks. The binary doesn't matter much, but you should prefer ones with debugging symbols, and if you want to call functions in a particular library, you should use a binary that is linked to that library.

Example session:

~~~ c
~% gdb ./test
(gdb) break main
Breakpoint 1 at 0x8048452
(gdb) run
Starting program: /home/pcl/sandbox/test
Breakpoint 1, 0x08048452 in main ()
(gdb) set $a = malloc(1234)
(gdb) call sprintf($a, "Hello %d", 12345*12345*12345)
$1 = 15
(gdb) print (char*)$a
$2 = 0x96c6008 "Hello 170287977"
(gdb) print (unsigned int)atoi("-1")
$3 = 4294967295
(gdb) print (unsigned int)atoi("4294967295")
$4 = 2147483647
~~~

gdb lets you use arbitrarily-named, untyped convenience variables, as you can see in the example. The only practical difference between `print $var = expr`, `call $var = expr`, and `set $var = expr` seems to be that set does not additionally assign the result to a history variable. Obviously you also have the full debugging facilities of gdb available as well.

It is also possible to do this on stripped binaries with no 'main' function, but there are many disadvantages:

~~~ c
~% gdb `which echo`
(gdb) inf files
	Entry point: 0x8048be0	0x08048154 - 0x08048167 is .interp
(gdb) break *0x8048be0
Breakpoint 1 at 0x8048be0
~~~

For a fully featured REPL for C, check out [c-repl](http://neugierig.org/software/c-repl/).
