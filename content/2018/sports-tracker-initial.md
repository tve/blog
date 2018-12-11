---
title: Ocean Sports Fitness Tracker initial status
date: 2018-07-15
thumbnail: "/img/sports-tracker/20180717_175701_sml.jpg"
project: sports-tracker
tags:
---

I've now taken the prototype on several outings and it's starting to
become useful. I'm learning how much I can display one the screen so
I can actually read it with a quick glance while paddling and bouncing
up&down in the ocean. It currently displays speed, min-max speeds over
the past 10 seconds, heading, heart rate, time, LoRa RSSI, as well as
number of stored trackpoints. Sorely missing is distance covered and to
do that properly will take some work.

![Tracker mount with velcro](/img/sports-tracker/20180714_153457_sml.jpg# fr w-40pct ml-3)
_The tracker is mounted with velcro on the footrest. The velcro mostly
works but is not really strong enough when plunging through a wave. The
yellow lanyard on the left saves the day in those cases._

The LoRa link has been giving me some trouble. It is working but not
quite at the distance I need. Right now, trackpoints are recorded to the
SPI flash chip and when radio connectivity is available the trackpoints
are sent. What is happening is that I don't have connectivity while
paddling and then on the drive home as I get closer it starts "spooling"
the recorded tracks. So everything is pretty much working except for the
fact that I don't quite have the range I need. I've narrowed the problem
down to two issues: I have poor antenna positioning in the waterproof
case (I knew that) and the anetenna isn't great (knew that too) but in
addition I'm loosing close to 10dB of RX sensitivity at my GW, which is
"not good". Put differently, both ends TX at the same power level and
RX SNR is almost 10dB worse at the GW end. I'm suspecting it's because
the radio is on a linux box (a CHIP), but it may well be something else.

From my LoRa GW packets are injected into Node-red where they're processed
and injected into the node-red-contrib-web worldmap. The code is currently
displaying the "real-time" position (except for the fact that there's
no real-time radio connectivity) as well as a track. One of the photos
attached here shows one outing of about 6.3mi I did with the tracker.

So far the proof of concept shows lots of promise and at the same time
there's lots left to do! Yay!

### Next steps

The thing that bugs me the most is the LoRa radio connection. I've tried
a better antenna (home made dipole) and I get connectivity using that,
but it's not great from an ergonomic point of view. And it made it clear
that if I could solve the 10dB RX sensitivity loss I could do with a
smaller antenna. So that's the next thing to fix.

One of the other tasks is to add code both in the tracker an in node-red
to smooth the track in order to be able to calculate distance covered
without over-inflating it due to GPS-induced zigzags. The keyword here
seems to be Ramer-Douglas-Peucker algorithm and some on-line variants
thereof.

The other task is to automatically determine the start and end of an
outing using some form of geofencing, and then storing the track of each
outing in a proper database so it can be processed in various manners and
also sent to endomondo or a similar sports tracking site automatically.

The [next post](/2018/spi-kills-lora) describes some radio problems with the prototype.
