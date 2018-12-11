---
name: driveway-chime
title: Driveway entrance chime powered by solar light barrier
date: 2018-10-06
thumbnail: "/img/light-barrier/20181005_lightbarrier_sq.jpg"
project: driveway-chime
categories:
- light barrier
- stm32
- solar
---
Solar powered light barrier across a driveway to signal the entrance and departure of vehicles and pedestrians
using a door chime. Communication between the light barrier and the chime is via RF links and MQTT.
<!--more-->

![View of the entrance](/img/light-barrier/20181005_145700.jpg# fr w-60pct ml-3)
_Driveway entrance with light barrier mounted, visible as small white PVC pipe pieces on far left
and far right._

The entrance to our property is about 500ft/150m away from our house and partially obscured by a
large oak tree. As a result we want some entrance chime that lets us know when someone arrives.
I set out to build a light barrier that transmits interruptions of the light barrier via RF to my
home automation system (built around MQTT and Node-Red). From there I can initiate the capture of
some images, send notifications to cell phones, and also ring a set of door chimes around the property.

The driveway is about 12ft/4m wide where the light barrier needs to go and there are a number of
trees that obscure the sunlight at various times of the day, often causing rapidly moving shade
patterns. Thus the light barrier has to be good enough to filter out sunlight as well as rapid light
changes.

# Failed attempts

My first attempt was to use a laser and photo sensor. I rather quickly abandoned that when I
realized how tiny the laser dot is at 12ft/4m distance. I'd have to pour some really solid concrete
to mount the laser so well that no matter how much wind or rain we have it doesn't move away from
the target. Plus every time I'd touch it I would have to fiddle to get it back on target. Not
appealing at all.

I then switched to an IR LED and IR detector. That worked at night but during the day the sunlight
just drove the thing crazy. There is so much light scattered in all directions and it changes so
much with wind that I could not establish any reasonable threshold.

# IR pulse light barrier

I finally searched on how one does this (would have been too easy to do this initially, wouldn't
it?). The answer I came up with is to pulse the IR LED at a specific frequency and use a detector
that is tuned to that frequency. This way it can filter out ambient IR light.

The [next post](/2018/light-barrier) describes the light-barrier.
