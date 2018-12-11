---
name: low-power-wifi-intro
title: Explorations into low-power wifi microcontrollers
date: 2018-11-18
thumbnail: "/img/low-power-wifi/20181119_231556_sq.jpg"
project: low-power-wifi
tags:
- low-power
- wifi
---
Operating Wifi at low power is difficult, Wifi is just way more complex and chatty than simple RF
links. This project and blog post series explores what is available at the end of 2018.
<!--more-->

In this exploration, low-power refers to a wifi-connected microcontroller that runs at least for a week on a 1000mAh
LiPo and could thus run indefinitely on a small 2"x3"/5x8cm 6V 100mA solar panel with good
sun exposure.

I looked into using the esp8266 on battery in 2015 when I first started using it and quickly gave
up.
At the time it became apparent that while the hardware could sleep at low power, the software was
not capable of keeping the lid on the power consumption.
I soon gave up and relegated the esp8266 to powered projects.
I recently decided to take another look at not only the esp8266 but also the esp32 to see whether
things have improved. I looked at two approaches.

![Espressif esp8266 RF node and 2600mAh Li-Ion cell](/img/low-power-wifi/20181119_231556.jpg# fr
w-50pct ml-3)

The first approach is to build systems that spend most of their time in deep sleep mode and only
wake-up periodically to connect to Wifi. In deep sleep mode all RAM contents is lost, so the system
boots from scratch each time it wakes up. It's really as if someone briefly flipped the on/off
switch to the on position every now and then.

This type of periodic wake-up systems are well matched to sensor nodes
that can spend most of their time asleep with the Wifi
turned off and occasionally wake up, take a measurement, briefly connect to Wifi to
transmit their data and then sleep again.
In those systems the power calculation revolves around how often the Wifi has to be turned on
and how much power it takes to connect and send out a measurement.
There are a good number of projects that successfully use esp8266s for such systems and
achieve a year of run-time on a small battery given a low-enough duty cycle.

The second approach to achieving low power Wifi is to make use of the power saving features of the
Wifi protocols and carefully manage when the processor and the radio are active to communicate over
Wifi and when they're asleep.
The benefit of this approach is that the systems give the appearance of always being on: they can be
reached at any time so one can send them commands and queries.

Somewhere in-between the always-on and mostly-asleep approaches there is a middle ground where it may
take a few seconds or a few minutes for a command to reach a system. This can be useful if, for
example, the command is a wake-up call and causes the system to stay awake for a short time period
so it can receive further commands or be used interactively.

I am using two use-cases to drive my investigation.
The first one is a wifi-serial bridge like [Esp-Link](http://github.com/jeelabs/esp-link) or an
STM32 Black-Magic-Probe debug tool.
I use them all the time in a powered mode in my lab but when I deploy low-power sensors in the
field I often find problems that I didn't spot before and it would be really convenient to be able
to deploy a serial-wifi bridge or debug probe for a couple of days or weeks on battery power to help
in the troubleshooting.

The second use-case is an RF-MQTT gateway (e.g. RFM69-MQTT or LoRa-MQTT).
It would be very helpful to deploy such a gateway on battery power 
as a stop-gap measure to fill-in an area with poor coverage
until I get real power installed.
Alternatively it would be awesome to be able to run a GW entirely on solar.

### A concrete target

What does it take to run a microcontroller on battery for days or weeks?
The following table provides a first-level calibration by showing run-time
using a 1000mAh battery vs. average power consumption
over a 0.01mA to 100mA range.

avg [mA] | life @1000mAh [days] | solar @4h/day [mA]
---:| --: | --:
100 | 0.4 | 600
10 | 4.2 | 60
1 | 41.7 | 6
0.1 | 416.7 | 0.6

The left column lists various average power consumptions in mA, the center column the number of
expected days of run-time on a 1000mAh battery, and the right column how large a solar panel
(6V nominal type) would be required to run indefinitely. These are just back-of-the-envelope
calculations that get into the right ballpark and that sweep voltage differences and other
details under the rug.

The goal then is find a way to stay connected to Wifi at an average power consumption of about
10mA. Using a relatively cheap 2500mAh 18650 Li-Ion cell such a device could plausibly run for
close to two weeks and using a 100mA solar panel with decent exposure it could plausibly run indefinitely.
Getting below 10mA would be really exciting because this would enable systems that run for about a
month on an 18650 Li-Ion cell, specially if one spends a bit more and gets a 3000mAh one...

The next question is how do low-power Wifi microcontrollers stack up against this goal?
The table below lists power consumption figures gleaned from datasheets (all at 3.3V) for
the Espressif esp8266, the Espressif esp32, and the RealTek RTL8710.

mode | esp8266 | esp32 | rtl8710 | comment
--- | ---:| ---:| ---:| ---
deep-sleep | 0.01mA | 0.01mA | 0.02mA | powered-down, memory lost, only wake-up timer running
sleep | 0.9mA | 0.8mA | 0.5mA | cpu stopped, timer running, memory&state preserved
cpu | 15mA | 20mA | 25mA | cpu running, RF off
rx | 56mA | 100mA | 62mA | receiving packet
tx | 320mA | 400mA | 162mA | transmitting (see note)

_Note: the esp32 datasheet says that the ~200mA TX power listed in its table 14 apply to
a 50% TX duty cycle, hence the 400mA listed here (and I can defintiely confirm that it consumes
that much using a scope). I also doubled the esp8266 power figure from its datasheet as I'm assuming
that this is also for a 50% duty cycle (I can also confirm seeing over 300mA myself)._

These numbers make it very clear that the systems must spend the majority of their time in one of
the two sleep modes. Running the processor all the time is not in the cards and any form of radio
activity must be limited and carefully managed.

The [next post](/2018/lp-wifi-esp8266-1) explores using the esp8266 in periodic wake-up mode
where it spends most of its time in deep-sleep and only wakes-up every now and then.
