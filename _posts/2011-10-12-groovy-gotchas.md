---
id: 372
title: Groovy Gotchas
date: 2011-10-12T23:16:32+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=372
permalink: /2011/10/groovy-gotchas/
categories:
  - programming
tags:
  - groovy
  - programming
  - ruby
---
## Map[&#8216;key&#8217;]

When calling putAt() on a Map with a String as a key, the Object version of putAt() is selected over the Map version. In other words, it will only call Map.put() if a property with the same name does not exist. However, calling getAt() on a Map will only call Map.get(), and will never return an object property.

<pre class="brush: groovy; title: ; notranslate" title="">class A extends HashMap { String y; } // any Map

a = new A();

a['x'] = 1;
a['x'];           // =&gt; 1
a.get('x');       // =&gt; 1
a.putAt('x', 2);
a.getAt('x');     // =&gt; 2

a['y'] = 1;
a['y'];           // =&gt; null
a.get('y');       // =&gt; null
a.putAt('y', 2);
a.getAt('y');     // =&gt; null
org.codehaus.groovy.runtime.DefaultGroovyMethods.putAt(a, 'y', 3)
a.getAt('y');     // =&gt; 3 !?

class B { String y = "hi"; }
(new B()).getAt('y');  // =&gt; "hi"
(new B())['y'];        // =&gt; "hi"

class C implements Map { String y; /* implement Map methods */ } 
(new C()).getAt('y');  // =&gt; null
(new C())['y'];        // =&gt; null
</pre>

## Ranges

The exclusive range operator (..<) generates a Range where the ending value is one step closer to the beginning value. The less-than symbol can be somewhat unintuitive for descending ranges.

<pre class="brush: groovy; title: ; notranslate" title="">enum E { ONE, TWO, THREE, FOUR }
((E.ONE)..(E.THREE)).toList()  // [ONE, TWO, THREE]
((E.ONE)..&lt;(E.THREE)).toList() // [ONE, TWO]
((E.THREE)..(E.ONE)).toList()  // [THREE, TWO, ONE]
((E.THREE)..&lt;(E.ONE)).toList() // [THREE, TWO]
</pre>

Indexing a List with a Range only considers `from` and `to` values _after_ the above adjustment; the result of Range.toList() is irrelevant, and whether or not it was an exclusive range is not known. Three steps are performed to get the result:

  1. Negative values are normalized to positive values by adding them to List.size()
  2. The result is generated from List.subList( min(from, to), max(from, to) )
  3. If from > to, the result is reversed

In Ruby, `a[0...-1]` means &#8220;From the 0th element up to and excluding the last element,&#8221; whereas the ostensibly equivalent construct in Groovy, `a[0..<-1]`, means &#8220;From the 0th element to the 0th element.&#8221;

<pre class="brush: groovy; title: ; notranslate" title="">a = [0,1,2,3,4]

a[0..-1]             // =&gt; [0, 1, 2, 3, 4]
a[0..-2]             // =&gt; [0, 1, 2, 3]
a[0..&lt;-1]            // =&gt; [0]
a[0..&lt;-2]            // =&gt; [0, 1, 2, 3, 4]

a[0..-2]             // =&gt; [0, 1, 2, 3]
a[-1..-2]            // =&gt; [5, 4]
a[-1..&lt;-2]           // =&gt; [5]
</pre>

Compare to Ruby:

<pre class="brush: ruby; title: ; notranslate" title="">a[0..-1]              # =&gt; [0, 1, 2, 3, 4]
a[0..-2]              # =&gt; [0, 1, 2, 3]
a[0...-1]             # =&gt; [0, 1, 2, 3]
a[0...-2]             # =&gt; [0, 1, 2]

a[0..-2]              # =&gt; [0, 1, 2, 3]
a[-1..-2]             # =&gt; []
a[-1...-2]            # =&gt; []
</pre>