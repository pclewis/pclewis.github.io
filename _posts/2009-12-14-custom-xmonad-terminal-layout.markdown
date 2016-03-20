---
author: pcl
comments: true
date: 2009-12-14 17:39:18+00:00
layout: post
permalink: /2009/12/custom-xmonad-terminal-layout/
slug: custom-xmonad-terminal-layout
title: Custom xmonad Terminal Layout
wordpressid: 5
categories:
- xmonad
tags:
- haskell
- xmonad
---

[![My terminal layout](http://blog.pclewis.com/wp-content/uploads/2009/12/xmonad-0e-300x225.jpg)](http://blog.pclewis.com/wp-content/uploads/2009/12/xmonad-0e.jpg)

I've been using [xmonad](http://xmonad.org/) as my window manager at work for a while now. xmonad is a minimal tiling window manager written in and configured using [Haskell](http://haskell.org/). The main stated advantage of using a tiling wm is that all your windows automatically expand to fill up your screen. What I like most about it is actually the ability to change workspaces (or virtual desktops) on each screen individually, without having to use separate X screens.

Previously, I had used xfce4 and compiz-fusion with all sorts of flashy eye candy, like the [rotating cube](http://www.youtube.com/watch?v=8JknAj1Vob8). I wanted to be able to rotate each monitor separately, which was wasn't possible if you had your desktop stretched across multiple monitors. You had to use separate X screens for each monitor, so windows had to stay on the monitor they were started on.

xmonad works exactly the way I want it to: I have one big desktop, I can move windows back and forth between monitors, and I can switch an entire workspace from one monitor to the other without having to do any kind of rearranging myself. Even if they use different resolutions. It's a big step down in terms of eye-candy and ease of configuration, but a big step up in productivity and efficiency.

One thing that always bothered me is that I didn't really use the tiling capabilities of xmonad at all. Most things I just use full screen: browser, email, editor, etc. I do use a handful of terminals, and those are usually what you see the most of in screenshots of tiling wms. Unfortunately, my screen is too wide for a single terminal but too narrow for two, so I just used a floating layout and manually scrambled them all over the place. There just isn't a way to make tiling work for me without having terminals overlapping or squished.

I tried several layouts, but nothing really fit and I always ended up reverting to the pure floating layout. Eventually, I noticed a pattern in the way I was arranging my floating terminals. I've always been impressed with how simple and short most of the xmonad source code is, so I decided I'd just write my own layout and be done with it.

How hard could it be?

<!-- more -->

Well, as it turns out, a lot harder than it has to be. I looked through a handful of the existing layouts trying to understand what in the world was going on when I found [LayoutBuilder](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-LayoutBuilder.html). I quickly came up with a layout that matched the way I was arranging my terminals manually:

~~~
layoutR 0.5 0.5 (relBox 0.00 0.00  0.75 0.98) Nothing
    (spacing 14 $ Column 1) $
layoutAll       (relBox 0.25 0.02  1.00 1.00)
    (spacing 14 $ Column 1)
~~~

The first two lines create a layout that takes the first half of the windows and arranges them in a column on the left 3/4ths of the screen. The last two lines take all remaining windows and places them in the right 3/4ths of the screen, slightly lower than the others. This way there is a gap between the terminals, and they overlap but leave the bottom line visible. Combined with transparency, this lets me keep an eye on or refer to terminals while I'm working in a different one.

The only problem is, the terminals on the left are always on top, and the terminals on the right are always behind them. There's no way to raise the terminals on the right without rearranging or floating them. Fortunately, with some help from the #xmonad irc channel, I was able to use [ToggleLayouts](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-ToggleLayouts.html) and a custom [LayoutModifier](http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Layout-LayoutModifier.html) to make a key toggle which side is in front:

~~~
data SwapTop a = SwapTop Int
  deriving (Show, Read)

instance LayoutModifier SwapTop a where
  pureModifier (SwapTop a) _ _ wrs =
    (reverse wrs, Just $ SwapTop a)

swapTop :: l a -> ModifiedLayout SwapTop l a
swapTop = ModifiedLayout (SwapTop 0)

myLayout =
  toggleLayouts (
    layoutR 0.5 0.5 (relBox 0.00 0.00  0.75 0.98) Nothing
        (spacing 14 $ Column 1) $
    layoutAll       (relBox 0.25 0.02  1.00 1.00)
        (spacing 14 $ Column 1)
  ) (swapTop (
    layoutR 0.5 0.5 (relBox 0.00 0.02  0.75 1.00) Nothing
        (spacing 14 $ Column 1) $
    layoutAll       (relBox 0.25 0.00  1.00 0.98)
        (spacing 14 $ Column 1)
  ))
~~~

The first two lines create a data type constructor for my new modifier, SwapTop. All the examples took a parameter, and ghc had a fit when I removed it, so SwapTop takes an integer for no reason. Line 6 is the entirety of the magic. pureModifier gets passed a list of (Window, Rectangle) pairs after the underlying layout is finished, and is expected to return a modified list and/or a new modifier. The windows are rendered in the order they appear in this list, so all I had to do in my case was reverse it. The swapTop function on line 9 just packages up my SwapTop data type in a ModifiedLayout data type.

Lines 17-21 define the alternate layout, with my SwapTop applied. I also swapped the y values, so the right side is the higher one and the bottom line of the now-obscured terminals is still completely visible. Originally I just wanted to make the focused window appear on top, but this way was simpler and turned out to suit my workflow a lot better.

Using an overlapping layout may make tiling wm purists cry, but it's been working extremely well for me, and now I no longer look like an idiot awkwardly shuffling around floating terminals all the time.
