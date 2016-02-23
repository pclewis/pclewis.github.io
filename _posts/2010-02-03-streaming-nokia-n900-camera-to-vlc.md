---
id: 244
title: Streaming Nokia N900 Camera to VLC
date: 2010-02-03T23:08:31+00:00
author: pcl
layout: post
guid: http://blog.pclewis.com/?p=244
permalink: /2010/02/streaming-nokia-n900-camera-to-vlc/
categories:
  - misc
tags:
  - n900
---
I recently had need to look at the back of my own head, and using the camera on my phone seemed like the easiest way to do it. I found [a guide](http://wiki.maemo.org/Streaming_video_from_built-in_webcam) on the Maemo wiki, but it was for the N800 and I didn&#8217;t have the hantro4200 encoder it was trying to use. After learning more than I ever wanted to about gstreamer and sdp files, I came up with a way that works for me.

In my setup, my computer is 192.168.0.100 and the phone is 192.168.0.200. You will have to replace them with your own IP addresses.

Here is the command to start gstreamer on the phone. You will probably want to put it in a script:

<pre class="brush: bash; title: ; notranslate" title="">gst-launch v4l2camsrc device=/dev/video0 ! \
           dsph264enc ! \
           rtph264pay ! \
           udpsink host=192.168.0.100 port=5434
</pre>

If <tt>gst-launch</tt> is not found, you probably need to install the <tt>gstreamer-tools</tt> package:

<pre class="brush: plain; light: true; title: ; notranslate" title="">apt-get install gstreamer-tools</pre>

To use the camera on the front of the phone, you can change the device to <tt>/dev/video1</tt>.

Here is the minimal sdp file I was able to use with VLC to get it to play. Using the &#8220;Open Network&#8221; dialog to try and play an RTP stream did not work.

<pre class="brush: plain; title: ; notranslate" title="">v=0
m=video 5434 RTP/AVP 96
c=IN IP4 192.168.0.200
a=rtpmap:96 H264/90000
</pre>

The second line (<tt>m=</tt>) contains the port, the third (<tt>c=</tt>) contains the IP address of the phone, and the fourth (<tt>a=</tt>) specifies the codec.

To use MP4 instead of H264, you can just change <tt>h264</tt> to <tt>mp4v</tt> everywhere. In the SDP file, it should be <tt>MP4V-ES</tt>, as in: <tt>a=rtpmap:96 MP4V-ES/90000</tt>. If you get errors in VLC like:

<pre class="brush: plain; light: true; title: ; notranslate" title="">avcodec warning: cannot decode one frame (14922 bytes)</pre>

Then add <tt>send-config=true</tt> to the <tt>rtpmp4v</tt> part of the gstreamer pipeline, and make sure you start VLC before you start streaming:

<pre class="brush: bash; title: ; notranslate" title="">gst-launch v4l2camsrc device=/dev/video0 ! \
           dspmp4venc ! \
           rtpmp4vpay send-config=true ! \
           udpsink host=192.168.0.100 port=5434
</pre>

For H263, you can try <tt>dsph263enc</tt>, <tt>rtph263pay</tt> and <tt>H263-1998</tt> or <tt>H263-2000</tt>, but I couldn&#8217;t get it to work.

I don&#8217;t know if there&#8217;s a way to control the focus, white balance, etc, but I was able to use the flashlight-applet to turn on the camera LEDs while streaming after I downgraded to 0.2-0:

<pre class="brush: plain; light: true; title: ; notranslate" title="">apt-get install flashlight-applet=0.2-0</pre>