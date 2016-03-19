---
layout: post
title: XMonad, Spacemacs Style
---

## Background

I've been using [XMonad] for almost 10 years but I barely know my own key bindings. Every time I open `xmonad.hs` I find bindings I don't remember setting, doing things I'd forgotten were even possible. I'm hesitant to add new bindings because I know I'll forget them.

As a recent convert to [Spacemacs], one of the things I love most is that the key bindings are mnemonic key _sequences_, with help that pops up when you hesitate along the way.

I want the same thing in XMonad.

## Making It So

### Key Sequences

The first problem was figuring out how to bind key sequences instead of key presses. I searched for things like "xmonad key sequence," "xmonad multiple key bind," "xmonad prefix key," and so on, until finally [a blog post by Alexander Kojevnikov][1] pointed me in the right direction.

[XMonad.Actions.Submap] lets you bind a key to the `submap` action, which accepts another map of key bindings. When called, `submap` intercepts the next keypress and dispatches accordingly.

### Even Better: Emacs-style Bindings

From Submap I stumbled onto [XMonad.Util.EZConfig], which makes the key binding syntax much nicer. Instead of specifying a tuple with the modifier bitmask and appropriate `xK_` variable, you just use a string with an Emacs-like notation. It even supports specifying a sequence of keys, so you don't need to use Submap directly.

~~~ haskell
-- Using only Submap
((modm, xK_x), submap . M.fromList $ [((0, xK_w), spawn "xmessage 'woohoo!'")])

-- With EZConfig
("M-x w", spawn "xmessage 'woohoo!'")
~~~

### Visual Feedback

Submap and EZConfig don't provide any kind of feedback that you've entered a submap, or expose any way to add your own. We can use `>>` to call `spawn` before `submap`, but `submap` blocks waiting for input; `dmenu` won't run because it can't grab the keyboard, and `xmessage` won't appear because XMonad won't notice the new window until after `submap` exits.

Luckily, [`dzen2`] works and is perfect for what I have in mind. We just have to go back to calling `submap` manually:

~~~ haskell
mySubmap = submap . M.fromList $
  [((0, xK_w), spawn "xmessage 'woohoo!'")
  ]

main = xmonad $ defaultConfig `additionalKeys`
                [ ("M-m", spawn "echo Entered Submap | dzen2 -p 2"
                       >> mySubmap) ]
~~~

With no parameters, `dzen2` closes when its input is closed. We can use `-p 2` to keep it open for 2 seconds, but what we really want is for it to _wait_ a second before appearing, and to disappear when we exit the submap. With `spawnPipe` from [XMonad.Util.Run] we can get a handle that we can write to and close with `System.IO`:

~~~ haskell
mySubmap = submap . M.fromList $
  [((0, xK_w), spawn "xmessage 'woohoo!'")
  ]

dzen message action = do
  handle <- spawnPipe "sleep 1 && dzen2"
  io $ hPutStrLn handle message
  action
  io $ hClose handle
  
main = xmonad $ defaultConfig `additionalKeys`
                [ ("M-m", dzen "In Submap" mySubmap]) ]
~~~

(`io` is necessary to lift the IO monad into the X monad, because Haskell)

### Parsing the Keymap

Now we can pop up a message, how do we make it tell us what the keys do?

[XMonad.Util.NamedActions] is an option, but it looks really cumbersome. I don't want to have to think of documentation and keep it up to date. It might be possible to do something with [Template Haskell][2] to auto-generate names for actions, but that seems like too much work.

Since I keep my bindings neatly formatted anyway, the easiest thing to do is just yank the keys and actions directly out of xmonad.hs.

I know my submaps are always going to start with a line like `submapName =` and end with a `]` on a line by itself. [`awk`] makes it easy to grab a range of lines:

~~~ bash
~$ cat ~/.xmonad/xmonad.hs | awk '/^mySubmap/,/]$/'
mySubmap = submap . M.fromList $
  [((0, xK_w), spawn "xmessage 'woohoo!'")
  ]
~~~

With a little [regular expression][3] magic, [`sed`] can extract the import parts:

~~~ bash
~$ !! | sed -r '1d;$d;s/.*xK_(.)\), (.*)\)$/\1 -> \2/'
w -> spawn "xmessage 'woohoo!'"
~~~

(`-r` swaps which kind of paren we need to quote, and `1d;$d` trims the first and last line)

With a bit of work you could do this with either `sed` or `awk` alone, but I think this way is simpler.

Later, once we have a bunch of them, [`column`] can wrap them into columns:

~~~ bash
~$ for i in {1..6}; do !!; done | column
w -> spawn "xmessage 'woohoo!'"	w -> spawn "xmessage 'woohoo!'"
w -> spawn "xmessage 'woohoo!'"	w -> spawn "xmessage 'woohoo!'"
w -> spawn "xmessage 'woohoo!'"	w -> spawn "xmessage 'woohoo!'"
~~~

Now instead of calling `dzen2` directly, we can call a shell script that contains this command pipeline:

~~~ bash
#!/bin/sh
KEYMAP=$1
( echo "$KEYMAP"                                   \
; cat ~/.xmonad/xmonad.hs                          \
  | awk '/^'"$KEYMAP"'/,/]$/'                      \
  | sed -r '1d;$d;s/.*xK_(.)\), (.*)\)$/\1 -> \2/' \
  | columns | expand
; cat ) | dzen2 -l 5 -e onstart=uncollapse
~~~

A few things to note:

- `columns` outputs tab characters, which don't work in dzen2. `expand` converts them to spaces.
- We need to run the pipeline in a subshell that ends with `; cat` to connect `dzen2`'s input channel to the output handle we have in XMonad.
- To make dzen2 show multiple lines, we need the `-l` parameter. This makes the first line the title, and the rest show up in an additional window that is hidden by default unless we add `-e onstart=uncollapse`. 

### Positioning

The last piece of the puzzle is getting the help to popup on the bottom of whatever monitor has focus, and `dzen2` doesn't seem to have any easy way to do this. It does make it easy to manually set the location and dimensions, we just need to figure out what those are.

We can get the location and dimensions of the focused screen in XMonad by stealing some code from `floatLocation` in [XMonad.Operations]:

~~~ haskell
windowScreenSize :: Window -> X (Rectangle)
windowScreenSize w = withDisplay $ \d -> do
    ws <- gets windowset
    wa <- io $ getWindowAttributes d w
    bw <- fi <$> asks (borderWidth . config)
    sc <- fromMaybe (W.current ws) <$> pointScreen (fi $ wa_x wa) (fi $ wa_y wa)

    return $ screenRect . W.screenDetail $ sc
  where fi x = fromIntegral x

focusedScreenSize :: X (Rectangle)
focusedScreenSize = withWindowSet $ windowScreenSize . fromJust . W.peek
~~~

We can then pass those to our script with something like:

~~~ haskell
keyMapDoc :: String -> X Handle
keyMapDoc name = do
  ss <- focusedScreenSize
  handle <- spawnPipe $ unwords ["~/.xmonad/showHintForKeymap.sh", name, show (rect_x ss), show (rect_y ss), show (rect_width ss), show (rect_height ss)]
  return handle
~~~

The math is straightforward so I won't cover it here. We end up with something like:

~~~ bash
LINE_HEIGHT=14
INFO=$(cat ...)
N_LINES=$(wc -l <<< "$INFO")
OFFSET=$((LINE_HEIGHT * N_LINES))
Y=$((BOTTOM - OFFSET))
(echo "$KEYMAP"; echo "$INFO"; cat) | dzen2 -h $LINE_HEIGHT -y $Y ...
~~~

## The Result

![Screenshot](/img/2016-03-19-xmonad.png)

It's not quite nice enough to start a project called `spacenads` yet, but it's exactly what I wanted. I have made some more tweaks and gone a little further than is described here, but you can see what I'm actually using on [my github][4].



[XMonad]: http://xmonad.org
[Spacemacs]: https://github.com/syl20bnr/spacemacs
[XMonad.Actions.Submap]: http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Actions-Submap.html
[XMonad.Util.EZConfig]: http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Util-EZConfig.html
[XMonad.Util.Run]: http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Util-Run.html
[XMonad.Util.NamedActions]: http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Util-NamedActions.html
[XMonad.Operations]: http://xmonad.org/xmonad-docs/xmonad/src/XMonad-Operations.html
[`dzen2`]: https://github.com/robm/dzen
[`awk`]: http://man7.org/linux/man-pages/man1/gawk.1.html
[`sed`]: http://man7.org/linux/man-pages/man1/sed.1.html
[`column`]: http://man7.org/linux/man-pages/man1/column.1.html
[1]: http://kojevnikov.com/xmonad-metacity-gnome.html
[2]: https://wiki.haskell.org/Template_Haskell
[3]: http://gnosis.cx/publish/programming/regular_expressions.html
[4]: https://github.com/pclewis/dotfiles/tree/master/xmonad/.xmonad
