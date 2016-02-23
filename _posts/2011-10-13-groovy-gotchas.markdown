---
author: pcl
comments: true
date: 2011-10-13 03:16:32+00:00
layout: post
permalink: /2011/10/groovy-gotchas/
slug: groovy-gotchas
title: Groovy Gotchas
wordpressid: 372
categories:
- programming
tags:
- groovy
- programming
- ruby
---

## Map['key']




When calling putAt() on a Map with a String as a key, the Object version of putAt() is selected over the Map version. In other words, it will only call Map.put() if a property with the same name does not exist. However, calling getAt() on a Map will only call Map.get(), and will never return an object property.



~~~ groovy
class A extends HashMap { String y; } // any Map

a = new A();

a['x'] = 1;
a['x'];           // => 1
a.get('x');       // => 1
a.putAt('x', 2);
a.getAt('x');     // => 2

a['y'] = 1;
a['y'];           // => null
a.get('y');       // => null
a.putAt('y', 2);
a.getAt('y');     // => null
org.codehaus.groovy.runtime.DefaultGroovyMethods.putAt(a, 'y', 3)
a.getAt('y');     // => 3 !?

class B { String y = "hi"; }
(new B()).getAt('y');  // => "hi"
(new B())['y'];        // => "hi"

class C implements Map { String y; /* implement Map methods */ } 
(new C()).getAt('y');  // => null
(new C())['y'];        // => null
~~~




## Ranges





The exclusive range operator (..<) generates a Range where the ending value is one step closer to the beginning value. The less-than symbol can be somewhat unintuitive for descending ranges.



~~~ groovy
enum E { ONE, TWO, THREE, FOUR }
((E.ONE)..(E.THREE)).toList()  // [ONE, TWO, THREE]
((E.ONE)..<(E.THREE)).toList() // [ONE, TWO]
((E.THREE)..(E.ONE)).toList()  // [THREE, TWO, ONE]
((E.THREE)..<(E.ONE)).toList() // [THREE, TWO]
~~~




Indexing a List with a Range only considers `from` and `to` values _after_ the above adjustment; the result of Range.toList() is irrelevant, and whether or not it was an exclusive range is not known. Three steps are performed to get the result:




  1. Negative values are normalized to positive values by adding them to List.size()


  2. The result is generated from List.subList( min(from, to), max(from, to) )


  3. If from > to, the result is reversed







In Ruby, `a[0...-1]` means "From the 0th element up to and excluding the last element," whereas the ostensibly equivalent construct in Groovy, `a[0..<-1]`, means "From the 0th element to the 0th element."



~~~ groovy
a = [0,1,2,3,4]

a[0..-1]             // => [0, 1, 2, 3, 4]
a[0..-2]             // => [0, 1, 2, 3]
a[0..<-1]            // => [0]
a[0..<-2]            // => [0, 1, 2, 3, 4]

a[0..-2]             // => [0, 1, 2, 3]
a[-1..-2]            // => [5, 4]
a[-1..<-2]           // => [5]
~~~

Compare to Ruby:

~~~ ruby
a[0..-1]              # => [0, 1, 2, 3, 4]
a[0..-2]              # => [0, 1, 2, 3]
a[0...-1]             # => [0, 1, 2, 3]
a[0...-2]             # => [0, 1, 2]

a[0..-2]              # => [0, 1, 2, 3]
a[-1..-2]             # => []
a[-1...-2]            # => []
~~~
