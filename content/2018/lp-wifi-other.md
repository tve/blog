---
title: MediaTek and RDA WiFi Microcontrollers
date: 2018-12-16
#thumbnail: "/img/low-power-wifi/rtlduino.jpg"
project: low-power-wifi
tags:
- low-power
- wifi
---

Unisoc's RDA5981 and MediaTek's MT7697 microcontrollers (as well as variants of these two) also
incorporate a WiFi radio on the chip and target low-power embeded IoT applications.
<!--more-->

### Unisoc RDA5981

![Hi-Link HLK-M50 module with RDA5981](/img/low-power-wifi/RDA5981-HLK-M50.jpg# fr w-30pct ml-3)
The RDA5981 incorporates an ARM Cortex-M4 core with 352KB RAM and 1MB flash in one chip.
The HLK-M50 is one low-cost module that incorporates the chip and I'm expecting one in the mail
soon. In the meantime I've taken a closer look at the
[datasheet](https://github.com/RDA5981/RDA5981/blob/master/Datasheet/RDA5981A%20DataSheet%20V1.5.pdf)
to see what I might expect.

One interesting feature of the chip is that its recommended supply voltage range is from 3.3V to
4.2V, which means that one can run it straight off a LiPo without regulator!
A first glipse at what to expect in terms of power consumption is provided by the following table
from the datasheet:

![RDA5981 power consumption](/img/low-power-wifi/rda5981-power-consumption.png# w-60pct ml-3 cb
nocaption)

The deep-sleep, RX and TX mode power levels look very comparable to the other microcontrollers
examined so far. I don't know what the "WIFI OFF" refers to but I wonder whether it's really
supposed to read 22 _milliamperes_ with the CPU running and the RF section being off.

Another module which uses the RDA5891 is MxChip's EMW3080 and [A_D
Electronics](https://adelectronics.ru/2018/03/04/%D0%BE%D0%B1%D0%B7%D0%BE%D1%80-%D0%BC%D0%BE%D0%B4%D1%83%D0%BB%D1%8F-rda5981x1/)
uncovered a more interesting power consumption table in its docs:

![EMW3080 power consumption](/img/low-power-wifi/rda5981-emw3080-power-consumption.png# w-80pct ml-3
center nocaption)

Is this table the power consumption while Wifi is connected is listed at 47mA, which certainly does
not compete with the Espressif microcontrollers or the RTL8710.
But as seen with the other microcontrollers, until the thing is running some test apps on the bench
it's impossible to tell what the actual power consumption will be.

### MediaTek MT7697

![LinkIt 7697 board with MT7697](/img/low-power-wifi/MT7697-LinkIt.jpg# fr w-30pct ml-3)
The MT7697 is a little different from all the other microcontrollers examined so far in that it
includes a separate processor for the Wifi portion. The application processor is an ARM Cortex-M4
with on-board RAM and a Flash interface. The Wifi processor is unspecified but some of the diagrams
show that it's also an ARM processor. While this dual-core set-up might look similar to the ESP32 it
differs in that the two microprocessors are completely separated and the Wifi processor is not
accessible to application code at all.

One of the benefits of the dual-processor architecture is that the application processor can be
completely shut down when sleeping while keeping the Wifi processor active to maintain a connection
to the access point. The downside is that there's more to power when both are on.

The most interesting document I've found so far is the
[power mode developer's guide](https://s3-ap-southeast-1.amazonaws.com/mediatek-labs-imgs/downloads/73f2af6bc5e82c6be0d533fdfedb4527.pdf)
which shows a 9.6mA sleep mode using FreeRTOS tickless idle and a 1.1mA "legacy sleep" mode:

![MT7697 Legacy sleep chart](/img/low-power-wifi/mt7697-legacy-power-chart.png# w-70pct center nocaption)

Another interesting [document on low power
modes](https://docs.labs.mediatek.com/resource/mt7687-mt7697/en/tutorials/low-power-mode-and-power-measurement)
has the following table:

Scenario | Typical Results [mA]
--- | ---:
Legacy Sleep | 0.667
WiFi Radio off (tickless) | 10.79
WiFi Radio off (legacy sleep) | 0.97
WiFi Connected (DTIM 1, tickless) | 14.18
WiFi Connected (DTIM 1, legacy cleep tickless) | 4.64
WiFi Connected (DTIM10, legacy sleep tickless) | 2.07

That looks right in the ballpark of the esp32 to me, but only experimentation will tell for sure...

Stay tuned for the next post...
