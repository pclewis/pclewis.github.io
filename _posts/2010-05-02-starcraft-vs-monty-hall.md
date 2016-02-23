---
id: 322
title: Starcraft vs Monty Hall
date: 2010-05-02T17:46:11+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=322
permalink: /2010/05/starcraft-vs-monty-hall/
categories:
  - misc
tags:
  - javascript
  - starcraft
  - statistics
---
If you&#8217;re not familiar with the [Monty Hall problem](http://en.wikipedia.org/wiki/Monty_Hall_problem), it goes something like this:

There are three doors, and one of them has a prize. You choose one of the doors, and Monty opens one of the others that is not a winner. Now you have the option to stick with your original choice, or switch to the remaining door. It might seem counter-intuitive, but switching doubles your odds of winning.

Some people have attempted to apply the same logic to scouting as [Zerg](http://starcraft.wikia.com/wiki/Zerg) using an overlord and a drone in [Starcraft](http://en.wikipedia.org/wiki/Starcraft). Stated similarly, the problem goes like this:

There are three starting locations, and one of them has your enemy base. You send a drone to one location, and your overlord gets to one of the other locations and discovers no enemy base. Now you have the option to stick with your original choice, or send your drone to the remaining location.

Sounds the same, but in this case, switching has no effect on your odds of winning. It is the same as the &#8220;[Monty Fall](http://probability.ca/jeff/writing/montyfall.pdf)&#8221; or &#8220;Ignorant Monty&#8221; variant of the Monty Hall problem, where Monty opens a door completely at random rather than one which is a non-winner.

The difference is because in the classic Monty Hall problem, you are initially choosing one door, which gives you 1/3 odds of your first choice being right. If you could choose to switch to _both_ other doors, you&#8217;d obviously have a 2/3 chance of winning. In fact, this is _exactly_ what you are doing when you switch, even after one of the doors has been revealed.

In the Starcraft problem, you are choosing _two_ locations to begin with, which gives you a 2/3 chance of being right. If you choose to switch your drone to the remaining location at any point, the overlord still has a 1/3 chance and the drone still has a 1/3 chance. In the Monty Hall problem, you switch from only an unknown door (1/3), to the empty door and an unknown door (2/3). In the Starcraft problem, you switch from both an empty location and an unknown location (2/3), to the same empty location and a different unknown location (2/3).

To demonstrate this visually, I&#8217;ve made a [simulator](http://blog.pclewis.com/scvsmh/) in Javascript.

**Update (5/3):**
  
I added a &#8220;Monty Hall mode&#8221; to the simulator, the implementation of which may help make this even clearer. Normally, I choose two random locations from the possible enemy starting points, and send the overlord to the first and the drone to the second. This leads to 3 possibilities with an equal chance of occuring: the overlord finds the base, the drone finds the base, or neither finds the base. Only in the final case, which occurs 1/3 of the time, would it be correct to switch. In &#8220;Monty Hall mode,&#8221; the overlord is not allowed to find the base &#8211; so the 1/3 of the time where the overlord would normally have found the base, is now added to the 1/3 of the time when neither finds the base, making it correct to switch 2/3s of the time.

I also added a &#8220;Stupid Overlord mode&#8221; which demonstrates that the order is important. If the overlord chooses an empty base first, rather than the drone choosing first, the chances of the drone being correct are (obviously) 1/2.
  
<!--more-->