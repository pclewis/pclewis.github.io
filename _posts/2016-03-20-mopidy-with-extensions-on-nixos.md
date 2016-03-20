---
layout: post
title: Mopidy with Extensions on NixOS
excerpt: How to get extensions working with Mopidy on NixOS.
tags: 
- nixos
---

[Mopidy] is a sweet [mpd]-compatible music player that can play local files and stream from services like Spotify and YouTube.

[NixOS] has [a package for Mopidy][1] and several extensions. You can install them with [`nix-env -i`], but Mopidy will not detect the extensions, and it can difficult to figure out why.

This is Nix working [the way it is supposed to work][2]: the extensions are Python libraries, and the build environment for the mopidy package can't be altered by installing other packages.

The solution is to make a new environment that depends on the mopidy package and any extension packages you want.

It's easy with [`nix-shell`]: 

~~~ bash
nix-shell -p mopidy mopidy-mopify mopidy-spotify --run mopidy
~~~

Turned into an executable script:

~~~ bash
#! /usr/bin/env nix-shell
#! nix-shell -p mopidy mopidy-mopify mopidy-spotify --run mopidy
~~~

There is also a [mopidy service], used with something like this in `/etc/nixos/configuration.nix`:

~~~ nix
services.mopidy = {
  enable = true;
  extensionPackages = [ pkgs.mopidy-spotify pkgs.mopidy-mopify ];
  configuration = ''
    [spotify]
    username = dade.murphy
    password = hack.the.planet
  '';
};
~~~

[Mopidy]: https://www.mopidy.com/
[mpd]: https://en.wikipedia.org/wiki/Music_Player_Daemon
[NixOS]: https://nixos.org/
[mopidy service]: https://github.com/NixOS/nixpkgs/blob/master/nixos/modules/services/audio/mopidy.nix
[`nix-env -i`]: https://nixos.org/nix/manual/#ch-basic-package-mgmt
[`nix-shell`]: https://nixos.org/nix/manual/#sec-nix-shell
[1]: https://github.com/NixOS/nixpkgs/blob/master/pkgs/applications/audio/mopidy/default.nix
[2]: https://nixos.org/nix/manual/#ch-about-nix
