---
name: lp-wifi-esp32-boards
title: Low-Power ESP32 Boards
date: 2018-12-01
thumbnail: "/img/low-power-wifi/esp32-boards-sq.jpg"
project: low-power-wifi
tags:
- low-power
- wifi
- esp32
---

_Updated 2019-08-25 to include EzSBC's ESP32-01 Breakout and Development Board_

In order to test the low-power performance of the esp32 I had to acquire a few boards and
I didn't really like any of them.
<!--more-->

The main characteristic I am looking for is the power consumption from a LiPo in deep sleep.
When I ordered the first board, I didn't think much about it. Oh, how wrong I was!
I then consulted Andreas Spiess' excellent [youtube](https://www.youtube.com/watch?v=-769_YIeGmI)
about battery powered esp32 boards, but most of them have frills I don't care for.
In the end, I'm concluding that I need to make my own minimal board, but first let's see how I got
there.

### NodeMCU Esp32S

![Esp32s NodeMCU board](/img/low-power-wifi/esp32-boards-nodemcu.jpg# fr w-30pct)
I ordered a NodeMCU Esp32s a while back because the price seemed right and it was one of these "I
may want to have this some day" type of purchases. I developed the first apps having it attached to
USB.

Then came the time to measure power consumption and for that I hooked up a LiPo so I don't have
a ground loop through the scope ground probes. To my shock the app didn't run! After quite some head
scratching I had the brilliant idea to measure the 3.3V rail. It showed only 2.3V!
(My LiPo was only ~30% full.) Ouch, that's a no-go. How can that be?

The answer is that there are LDOs and and LDOs...
The board uses an AMS1117 and its drop-out voltage, i.e., minimal input-to-output voltage drop is over
1V.
Calling this an LDO seems like a marketing stunt to me...
Anyway, probably a great board, but totally unsuitable for battery operation.

_Update:_ I somehow ended up with some more boards with an AMS1117 LDO and finally spent some time looking
whether I could replace it with a better one. Finding an LDO with the same package is easy but they
don't tend to have the same pin-out. I finally found the AP7361C-33ER, which has a low drop-out
and the same pin-out _and_ Digikey has it in stock! I used some hot air to desolder an AMS1117 and
put the AP7361C in its place and it works like a charm on a LiPo! Be careful when ordering, the "R"
at the end of the part number is key, it refers to the "reverse" pin-out that the AMS1117 uses.

### Adafruit Huzzah32

![Esp32 Huzzah32 board](/img/low-power-wifi/esp32-boards-huzzah32.jpg# fr w-50pct ml3)

I wanted to get to my battery power measurements badly so I loked around on Amazon for options with
prime shipping.
I saw the Adafruit Huzzah32 at $20, ouch! But with a LiPo connector and an integrated charger it's
clearly designed for battery operation. Ok, gotta pay for quality. Buy now!

The good news: it ran my first test apps fine. The bad news: it consumes power like there's no
tomorrow. Deep-sleep means 6.5mA. No, that's not a typo: milliamperes. Or put differently, a cool
1000x more than it should.

![CP2104 fix](/img/low-power-wifi/huzzah32-fix.jpg# fr w-50pct ml3)

At that point I wished I had ordered one of the cheap chinese boards. But I browsed around and found
the answer: the CP2104 USB-serial chip doesn't power down. More browsing later, I found that it can
be seduced into USB suspend state, i.e., low-power state, by adding a pull down resistor on the USB
D- line. That brought power consumption down into the 200uA-250uA range. Great improvement, but still far
from the goal. At least it's not totally ridiculous anymore.

I had another "fun" experience with the Huzzah32. Long story short it produced weird errors that I
couldn't reproduce with the NodeMCU. I eventually reached the hypothesis that the flash chip was not
working properly, so I decided to remove the esp32-wroom shield and replace the flash chip. Except
that I completely botched it and lifted two pads. Ouch. The result is a "creative" flash chip
placement visible in the first photo. It does work flawlessly now, though.

### Sparkfun Esp32 Thing

I continued browsing for what to buy, 'cause I started to get curious about whether there are any
low-power Esp32 boards with USB out there. If you reject all boards that have an AMS1117 LDO or that
have a CP2104/CP2102 USB interface then the pickings become real slim.

![Sparkfun Esp32 Thing board](/img/low-power-wifi/esp32-boards-thing.jpg# fr w-30pct)

The Sparkfun Esp32 Thing caught my eye. It has a different LDO and a FTDI USB chip. It also has a LiPo
charger and connector and seemed well designed. Again, $21. But that's better than $8 for a board
that doesn't even _run_ on LiPO. Buy now!

I plugged the Thing in and measured deep-sleep current: 2mA! Ok, that's unfair, it has a power-is-on
LED and a jumper-trace in the back that needs to be cut. After cutting the trace I measured 900uA.
At that point I got rather annoyed. This was the second expensive board that completely flunks the
deep-sleep test.

I browsed the Sparkfun docs: nothing about the power consumption, except for other users complaining
about the same thing. Some more browsing later I found out that another jumper-trace in the back
needs to be cut and switched around so the flash chip gets powered by a special LDO inside
the esp32 instead of the 3.3v rail.
This way it gets powered down in deep-sleep. Now the power consumption got down to 63uA. Looking at
the LDO data sheet reveals that its typical quiescent consumption is 55uA, so it's unlikely one can
get much lower.

The good news: it's better than the Huzzah32. The bad news: it's 10x more than the esp32's
deep-sleep consumption. In all fairness, the 55uA power consumption is good enough for the vast
majority of use-cases. Unless the esp32 is to sleep for hours between wake-ups the difference
between 63uA and 6.3uA does not matter much.

All-in-all this is the best board I've seen, except for the fact that it doesn't have an FCC certified
esp32 module. Not only does it have the best deep-sleep performance, it also has a well-engineered
usb/lipo power switchover/charge circuitry and it has a 32khz oscillator for accurate RTC timing.

### Building my own board

I haven't started yet but if I am to deploy battery-powered esp32's I want a board that:

- has no USB (it's convenient for prototyping but unnecessary in the field)
- has an esp32-wroom-32 or esp32-wrover module so the flash chip is properly connected and the whole
  thing is shielded
- has an LDO with a low drop-out voltage and low quiescent current.

About the LDO, the AP2112 used in the Sparkfun Thing is probably good enough. I looked at other
options and in the 500mA LDO category the Ti TLV755p has a 25uA quiescent current (Iq).
Stepping down to 300mA the Richtek RT9078 LDO has an Iq of 2uA and
the 300mA On semi NCP703 3 has an Iq of 12uA.
Using a 300mA LDO may be a bit tight depending on what else it's powering, but for the esp32 itself
it should be enough.

![EzSBC board](/img/low-power-wifi/esp32-boards-ezsbc.jpg# fr w-30pct)

### EzSBC ESP32-01 Breakout and Development Board

_This section was added 2019-08-25_ 

I became aware of this board through the designer's (Daniel deBeer) comments below and
decided to order two for evaluation. The boards are nicely done and the inclusion of 2 RGB LEDs is
welcome. The price is good and I had no difficulties running some of my test programs.

Long story short the boards use a TS9013 LDO and, true to Daniel's claims, I see a 13uA-15uA average
power consumption in deep sleep. I'd say: mission accomplished! I only hope he will have more in
stock soon so I can order a couple more...

Stay tuned for more posts about the Esp32's Wifi power consumption...

The [next post](/2018/lp-wifi-esp32-light) switches from deep-sleep to experiments using automatic
light-sleep allowing the esp32 to maintain an association with the AP while going into low-power
modes.
