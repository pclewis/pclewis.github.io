---
author: pcl
comments: true
date: 2010-02-04 04:08:31+00:00
layout: post
permalink: /2010/02/streaming-nokia-n900-camera-to-vlc/
slug: streaming-nokia-n900-camera-to-vlc
title: Streaming Nokia N900 Camera to VLC
wordpressid: 244
categories:
- misc
tags:
- n900
---

I recently had need to look at the back of my own head, and using the camera on my phone seemed like the easiest way to do it. I found [a guide](http://wiki.maemo.org/Streaming_video_from_built-in_webcam) on the Maemo wiki, but it was for the N800 and I didn't have the hantro4200 encoder it was trying to use. After learning more than I ever wanted to about gstreamer and sdp files, I came up with a way that works for me.

In my setup, my computer is 192.168.0.100 and the phone is 192.168.0.200. You will have to replace them with your own IP addresses.

Here is the command to start gstreamer on the phone. You will probably want to put it in a script:

~~~ bash
gst-launch v4l2camsrc device=/dev/video0 ! \
           dsph264enc ! \
           rtph264pay ! \
           udpsink host=192.168.0.100 port=5434
~~~

If gst-launch is not found, you probably need to install the gstreamer-tools package:

~~~ bash
apt-get install gstreamer-tools
~~~

To use the camera on the front of the phone, you can change the device to /dev/video1.

Here is the minimal sdp file I was able to use with VLC to get it to play. Using the "Open Network" dialog to try and play an RTP stream did not work.

~~~
v=0
m=video 5434 RTP/AVP 96
c=IN IP4 192.168.0.200
a=rtpmap:96 H264/90000
~~~

The second line (m=) contains the port, the third (c=) contains the IP address of the phone, and the fourth (a=) specifies the codec.

To use MP4 instead of H264, you can just change h264 to mp4v everywhere. In the SDP file, it should be MP4V-ES, as in: `a=rtpmap:96 MP4V-ES/90000`. If you get errors in VLC like:

`avcodec warning: cannot decode one frame (14922 bytes)`

Then add `send-config=true` to the rtpmp4v part of the gstreamer pipeline, and make sure you start VLC before you start streaming:

~~~ bash
gst-launch v4l2camsrc device=/dev/video0 ! \
           dspmp4venc ! \
           rtpmp4vpay send-config=true ! \
           udpsink host=192.168.0.100 port=5434
~~~

For H263, you can try dsph263enc, rtph263pay and H263-1998 or H263-2000, but I couldn't get it to work.

I don't know if there's a way to control the focus, white balance, etc, but I was able to use the flashlight-applet to turn on the camera LEDs while streaming after I downgraded to 0.2-0:

~~~ bash
apt-get install flashlight-applet=0.2-0
~~~
