---
id: 260
title: Analysis of Gemini Cybernetics CDS
date: 2010-03-24T23:13:55+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=260
permalink: /2010/03/analysis-of-gemini-cybernetics-cds/
categories:
  - second life
---
There have been some rumors going around about a new third-party system for Second Life. The system attempts to detect avatars using third-party clients capable of duplicating objects without the creator&#8217;s permission, and the rumors are that it uses some kind of QuickTime exploit or other nefarious means to actually examine the contents of your hard drive or otherwise invade your privacy without permission. I decided to take a quick look to see what it&#8217;s all about.

The system in question is called [GEMINI CDS Ban Relay](https://www.xstreetsl.com/modules.php?name=Marketplace&file=item&ItemID=2138424) and is advertised as a simple object which detects avatars entering your sim, and uses &#8220;a team of bots with special abilities&#8221; to determine if the avatar is &#8220;harmful.&#8221; If they are, it adds them to an external database, and can optionally ban or teleport them home. Entries in the database are permanent, so if an avatar has been considered harmful once, they are always considered harmful in the future. It claims to use several frequently updated methods to detect &#8220;illegitimate&#8221; clients.

The most obvious detection method, and the only one I discovered, is a script that triggers as soon as you enter a protected sim and [tells your client](http://wiki.secondlife.com/wiki/LlParcelMediaCommandList) to load up a special media URL. Using a tool like [Wireshark](http://www.wireshark.org/) or [ngrep](http://ngrep.sourceforge.net/), it is trivial to watch the HTTP request.

<!--more-->

<img class="size-medium wp-image-262 aligncenter" title="pcap1" src="http://blog.pclewis.com/wp-content/uploads/2010/03/pcap1-300x203.jpg" alt="Packet Capture" width="300" height="203" />

Broken down, the requested URL in my case was:

<pre>http://media.syscast.net/youtube.php
  ? <strong>licensekey</strong> = KBVaQkxGH1lDVRdBWA1GVEdaTFpQF1ReWUcREU9YEFxBRgxE
  & <strong>title</strong>      = BEYeQR8TAxxOLE8eBk0T
  & <strong>licensedon</strong> = B1IAQhUG
  & <strong>tvowner</strong>    = eBVbGUdLQFxeVA%3D%3D
  & <strong>videoid</strong>    = eEJVE0xDHwxZAUQSWBJFARMJV1RTE19BWUETGBMNQg0%3D</pre>

At a glance, the values are obviously all Base64 encoded &#8212; the trailing <tt>%3D</tt>s on the last two fields are a dead giveaway. Decoding them doesn&#8217;t produce anything human-readable, though; one online service gives me &#8220;(ZBLFYCUAXFTGZLZPT^YGOX\AF D&#8221; for the first field.

It&#8217;s easy to decode them in Ruby, where we can play with them a little more:

<pre class="brush: ruby; light: true; title: ; notranslate" title="">irb(main):002:0&gt; CIPHERTEXT = Base64.decode64('KBVaQkxGH1lDVRdBWA1GVEdaTFpQF1ReWUcREU9YEFxBRgxE')
=&gt; "(&#92;&#48;25ZBLF&#92;&#48;37YCU&#92;&#48;27AX\rFTGZLZP&#92;&#48;27T^YG&#92;&#48;21&#92;&#48;21OX&#92;&#48;20\\AF\fD"

irb(main):003:0&gt; CIPHERTEXT.length
=&gt; 36
</pre>

36 bytes is the same length as a [UUID](http://en.wikipedia.org/wiki/UUID) in canonical form, so it&#8217;s a pretty reasonable guess that this is my avatar&#8217;s UUID encrypted somehow. The only real encryption facility LSL makes available is [XORing Base64-encoded strings together](http://wiki.secondlife.com/wiki/LlXorBase64StringsCorrect). [XOR](http://en.wikipedia.org/wiki/Xor) has an interesting property: <tt>a ⊕ b = c</tt> ⇒ <tt>a ⊕ c = b</tt>; that is, if XORing some plaintext and some key produces some ciphertext, then XORing that ciphertext and the plaintext produces the key. Let&#8217;s give it a shot:

<pre class="brush: ruby; light: true; title: ; notranslate" title="">irb(main):004:0&gt; PLAINTEXT = 'a27b84f0-2757-4176-9579-43a181d4a5a0'
=&gt; "a27b84f0-2757-4176-9579-43a181d4a5a0"

irb(main):007:0&gt; CIPHERTEXT.bytes.each_with_index {|v,i| key &lt;&lt; (v ^ PLAINTEXT[i])}; key
=&gt; "I'm trying to replace msmtp with smt"
</pre>

That was easy enough. Using the key to decode the rest of the fields, we can see what is really being sent:

<pre>http://media.syscast.net/youtube.php
  ? <strong>licensekey</strong> = a27b84f0-2757-4176-9579-43a181d4a5a0
  & <strong>title</strong>      = Masakazu Kojima
  & <strong>licensedon</strong> = Numbat
  & <strong>tvowner</strong>    = 1269399503
  & <strong>videoid</strong>    = 1e8381fe7fdf727dce67632245c8dd6e</pre>

The second field (title) turns out to be my avatar name, followed by the sim name (licensedon), a UNIX time value (tvowner), and what looks like an MD5 hash (videoid). The time value is apparently used to prevent [replay attacks](http://en.wikipedia.org/wiki/Replay_attack): it is possible to immediately replay the request exactly and get a success response, but after about 30 seconds it causes an internal server error instead.

Visiting the parcel again to get another URL shows that only the time and MD5 hash change. Tampering with the values causes an immediate error redirect, which suggests that the MD5 hash is a signature to prevent forged messages. So, even though we could encrypt arbitrary values and send them, we&#8217;d need to know how the signature is generated for them to work.

The response from the server is innocent enough:

<pre>&lt;!--
--&gt;
&lt;html&gt;&lt;head&gt;&lt;/head&gt;&lt;body bgcolor="#7f7f7f" leftmargin="0" topmargin="0"&gt;&lt;img src="video-background.gif" width="2000px" height="2000px" border="0px" /&gt;&lt;/body&gt;&lt;/html&gt;</pre>

The video-background.gif file is just a transparent 1&#215;1 GIF image:

<pre>00000000  47 49 46 38 39 61 01 00  01 00 80 00 00 7f 7f 7f  |GIF89a..........|
00000010  00 00 00 21 f9 04 00 00  00 00 00 2c 00 00 00 00  |...!.......,....|
00000020  01 00 01 00 00 02 02 44  01 00 3b                 |.......D..;|</pre>

These are the only requests that are performed. Nothing nefarious appears to be taking place. There is no evidence of any kind of exploit, or the transmission of any kind of private information. So how does the service detect &#8220;illegitimate&#8221; clients?

The magic turns out to be in the &#8220;User-Agent&#8221; request header, which identifies the client. In my case: <tt>Mozilla/5.0 (Windows; U; Windows NT 6.0; chrome://navigator/locale/navigator.properties; rv:1.8.1.21) Gecko/20090305 SecondLife/Emerald Viewer (default skin)</tt>

By using [curl](http://curl.haxx.se/) to replay an old request, and simply replacing &#8220;Emerald Viewer&#8221; with the name of a random client from [the Onyx project](http://onyx.modularsystems.sl/viewer_reference.html) (NeilLife), I was able to get the system to ban an alternate account I created. Note that this worked even though the time value was old, and the HTTP response status was a 500 error, so it would appear that the system to prevent replay attacks is broken. Looks like they&#8217;re up to at least 1 false positive, even if it&#8217;s a technicality. Also note that the actual response body did not change, so there doesn&#8217;t seem to be any kind of exploit that is only sent to users of &#8220;bad&#8221; clients.

Using the same IP address and computer, I was able to go back to the same parcel on my main account with no trouble.

## Conclusions

Despite all the subterfuge, the Gemini CDS system seems to simply rely on &#8220;illegitimate&#8221; clients to identify themselves in an HTTP request. The message encryption is trivial to break, and it would seem that is only a matter of time before someone cares enough to figure out how to forge the message signature. It is trivial to avoid detection by this method, though the system may employ additional detection methods: the common way for third-party clients to identify each other is by the unique texture UUIDs they use for skin layer protection, which is passive and undetectable.

There is no evidence that the system uses any kind of exploit or other nefarious tactic. The system does not appear to record any data other than the avatar and client self-identification information.