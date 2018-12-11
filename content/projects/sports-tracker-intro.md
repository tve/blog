---
name: sports-tracker
title: Ocean Sports Fitness Tracker
date: 2018-07-14
thumbnail: "/img/sports-tracker/20180717_175701_crop.jpg"
project: sports-tracker
tags:
- stm32
- gps
- lora
---
Real-time GPS tracker for surfskiing (or kayaking, paddling, rowing)
with real-time display as well as LoRa and LTE radios for safety.
<!--more-->

The ocean sports fitness tracker's purpose is to track a paddler in the
ocean, provide real-time performance feedback to the paddler on a screen,
and to also provide real-time tracking to on-shore family or buddies for
safety purposes. After the outing the track is saved for performance
analysis and long term monitoring of progress, possibly using one of
the popular sports fitness tracking sites.

![Tracker on surfski on the ocean in Santa Barbara](/img/sports-tracker/20180717_175701_sml.jpg# fr w-40pct ml-3)
_Taking the tracker on a test spin out on the ocean off Santa Barbara._

### Background

The background for this project is that I recently started to surfski,
which is paddling a high-performance sit-on-top kayak in the open
ocean. The more wind and waves the better because that's where it is the
most fun: surfskis are designed specifically for surfing waves in the open
ocean. With this activity come several challenges, in particular safety
and tracking one's performance in order to learn and improve. Despite
first appearances, it's a very technical sport and there is a lot to
learn both about stroke technique as well as wave riding skill.

Most surfskiers use a GPS watch, e.g. a Garmin or similar, attached to
the foot strap. Looking at the prices and limitations of these things,
plus the fact that one ends up tied into some vendor's tracking platform
made me want to build my own. There are also a few more specialized
trackers, such as SpeedCoach or from Vaaka, but they're very pricey and
don't provide a real-time connection for someone on shore to track from
a safety point of view.

I tried to use an Android phone for many months and have concluded that
it just doesn't work. Right off the bat the battery life doesn't make
it. I use an excellent GPS tracking app designed to track family members
and it works beautifully as long as the phone screen is off and tucked
in my PFD. But it provides no real-time information about the speed
I'm going at, my heart rate, etc. If I use a sports tracking app that
displays this type of data and mount the phone so I can see it then the
battery lasts at best for an hour. Plus Android tries hard to terminate
the app to save battery. It has been an exercise in frustration!

### Goals

![Surfski with mounted tracker](/img/sports-tracker/20180714_153359_sml.jpg# fr w-30pct ml-3)

So I thought "how hard can it be?" :-). The minimal goals I set for the
tracker are the following:

- real-time GPS track
- heart rate monitor data capture
- display speed, time, distance, and heart rate in real-time on a
screen mounted to the footstrap
- transmit tracking info via LoRa and/or 4G LTE
- display progress on a web map with heart rate, current speed, etc
- waterproof enclosure for shallow immersion
- screen must be legible in full sunlight
- battery sufficient for 3 hour outings

I have some additional stretch goals:

- display min/max speed so I can see the acceleration going down a wave
and the slow-down hitting the back of one
- determine paddle stroke parameters, such as stroke rate, stroke
distance, etc. using either boat acceleration or a paddle mounted sensor
- determine wave parameters, such as wave height, speed frequency using an accelerometer

### Initial prototype

![Initial prototype](/img/sports-tracker/20180707_153359_sml.jpg# fr w-40pct ml-3)
_Inside the initial prototype. Transflective LCD at the top attached the inside of the lid.
GPS breakout and unused IMU breakout on top of the Jeenode Zero in the case body,
together with a suboptimal antenna. 18650 LiPo at the bottom with charger/protector
circuit attached to the left of the holder._

The initial prototype I build consists of a number of parts I cobbled
together from previous projects:

- Small stm32L082 board (Jeenode) with Hoperf RFM98 433Mhz LoRa radio
- Globaltop PA6H GPS on adafruit breakout board
- Bosch BNO055 compass, accelerometer, gyro on Adafruit breakout board
- Small protoboard tieing everything together and having a 16MB SPI flash chip mounted

In addition I added:

- Battery holder for an 18650, with small USB charger board
- Monochrome transflective ("sunlight readable") 128x64 LCD with SPI interface
- HM-11 bluetooth BLE radio to get the heart rate from a polaris 7 heart rate monitor
- Waterproof case with quick latch

I threw the prototype together so I could answer a number of questions:

- How well does the GPS track quick accelerations and decelerations?
(The Garmin phones tend to be really sluggish, the tracker in my Android
phone reliably records impossible speed bursts). I figured I might have
to try a couple of different GPS devices in order to find one that works
for the purpose.

- Can I make the LoRa link work? I live about 10 miles (16km) away from
the harbor where I usually paddle and at an altitude of 2100ft (about
650m) where I have line of sight. But that's quite a distance and starts
to cut into speed vs distance tradeoffs on the LoRa end. (Note that I'm
using LoRa "raw" and not LoRaWAN, which would not be suited due to the
data rate constraints it imposes.)

- Can I get one of the newer 4G LTE Cat-1M (or NB-IoT) radios to work? 
What do the costs look like as well as how does the physical integration
look like?

- Can I process the data using Node-Red and perhaps produce a decent UI in node-red.

- Can I make any productive use of the BNO055 or is that just an infinite rabbit hole?

The [next post](/2018/sports-tracker-initial) describes the initial build.
