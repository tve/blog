---
name: lp-wifi-esp32-1
title: Low-Power Wifi with the ESP32
date: 2018-11-26
thumbnail: "/img/low-power-wifi/esp32-set-up_sq.jpg"
categories:
- low-power-wifi
- low-power
- wifi
- esp32
---

The Esp32 is the successor of the Esp8266 by Espressif. It is much faster, has two cores, more
memory, more I/O, more everything. Does it also have more low-power Wifi mojo?<!--more-->

As I started to use the Esp32 for this project I ran into a number of snags. The first one was that
I tried the proprietary ESP-IDF framework and ran into issues with the unit resetting on me when I
tried light-sleep. I eventually switched to the Arduino framework although that comes with issues as
well. The software is not as ready for prime time as for the esp8266 as far as I can tell.

Another issue I encountered are the boards. In order to measure low currents I always power the
boards from LiPo so I don't end up with ground loop issues. I started with a NodeMCU board with an
AMS1117 voltage regulator but when I connected a LiPo to Vin it constantly reset due to power
brown-outs. I checked the 3.3V rail and indeed, not looking good. Pulling up the datasheet for the
AMS1117 I saw that the drop-out voltage @800mA is up to 1.3V. That's a new meaning for the "low" in
LDO...

I didn't fancy butchering the board so I ordered an Adafruit Huzzah32 ($20, ouch!) expecting it to
be well designed for battery operation given the LiPo charger and connector on the board and the
![Huzzah32 fix](/img/low-power-wifi/huzzah32-fix-annot.jpg# fr w-60pct)
_Huzzah32 fix: 47K from USB D- to GND_
brand name. Well, that turned into another disappointment! The way they have the CP1204 USB-serial
chip powered the chip remains active when USB is disconnected burning up 6.5mA! So in deep-sleep
mode the esp32 may well sip a couple of microamps but the CP1204 ruins it all by heating the room
up. Fortunately I'm not the first one to notice and I found a work-around, which is to connect a
47kOhm resistor between the USB D- and ground. This puts the CP1204 into suspend mode and then the
board consumes "only" about 250uA. Still frustrating. I can't figure out which other board is
better, some say the Wemos Lolin D32, but it's still more than 150uA and I'm not thrilled by the
CH340 USB-serial. In the end I'll have to make my own, I suppose...

With this prelude, my experimental set-up is basically the same as with the esp8266.

### Experimental set-up

![Experimental set-up](/img/low-power-wifi/esp32-set-up.jpg# fr w-40pct ml-3)

The set-up I use is as follows:

- an Adafruit Huzzah32 powered by a LiPo, modified to lower the deep-sleep current to 250uA
- a jumper hacked into the Huzzah32's battery ground connector to measure the supply current
- a uCurrent Gold to amplify the current (a 1 Ohm resistor would have probably done just fine given
  the magnitude of the currents)
- an oscilloscope to measure the current and a probe on the LED gpio pin to trigger the scope

The Wifi set-up is identical to the one used for the esp8266 tests.

So long for the preliminaries, the [next post](/lp-wifi-esp32-2) dives into the first experiments.

[Low-power Wifi series index](/categories/low-power-wifi)
