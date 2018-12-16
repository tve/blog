---
title: RTL8710 First Impressions
date: 2018-12-13
thumbnail: "/img/low-power-wifi/rtlduino.jpg"
project: low-power-wifi
tags:
- low-power
- wifi
- rtl8710
---

The RealTek RTL8710 is pitched directly at the esp8266 and it even comes in an RTL-00 module that has
the same pinout as an esp-12f module. How does it fare at low-power Wifi?

### Experimental set-up

![RTL-00 Module with an RTL8710AF ARM Cortex-M3](/img/low-power-wifi/rtlduino.jpg# fr w-50pct ml-3)
_RTLDuino with an RTL-00 Module which houses the RTL8710AF ARM Cortex-M3 Microcontroller.
The AMS1117 regulator it comes with is replaced by an AP7361C-33ER to enable powering with a LiPo._

To test the RTL8710 I purchased a NodeMCU-like development board shown in the photo on the right.
Like many similar esp8266 and esp32 boards it is ready to stick into a breadboard, has
a USB-serial interface, and a voltage regulator.

Unfortunately the RTLDuino comes with the dreaded AMS1777 "high-dropout regulator" that is unusable on a LiPo,
which I need to make accurate power consumption measurements. Fortunately after spending an hour
looking at the datasheets of all SOT223 LDOs I could find, I hit paydirt: the Diodes Incorporated
AP7361C-33ER is pin-compatible with the AMS1777, is a true LDO (360mV @1A typ), _and_ is stocked by
Digikey. (If you order one be sure to get the "ER" version and not the "E", which is not pin
compatible.) I used some hot air to swap the regulators out and so far it's working like a charm!

Another problem with the RTLduino is that it's not self-sufficient: the USB port is a bit of an
illusion because one can't program the thing through it. The processor in the RTL8710 is an ARM
Cortex-M3 and does not have any serial programming capability. Instead one has to resort to a
CMSIS-DAP, ST-Link, J-Link, or some JTAG programmer. I ended up using a bit of a hacked up set-up
with OpenOCD and an ST-Link-V2.

### Specifications

RealTek seems to have made an effort to make the RTL8710 (and the very similar RTL8795) accessible, but
between the language barrier and the ever-present secrecy the truth is that they're light years
behind Espressif. Documents are most readily available from the github repos of a number of russian
makers that discovered the module a couple of years ago. So it goes...

When I first saw the power specs in the datasheet I had to laugh. Here is the table I saw next to
the esp8266 table I'm quite familiar with. Copying? Naaah...

![RTL8710 Power Specifications](/img/low-power-wifi/RTL8710-power-specs.png# fl w-50pct mh0)
![ESP8266 Power Specifications](/img/low-power-wifi/Esp8266-power-specs.png# dib w-50pct mh0)

<p class="cb"></p>

The specs are basically the same, no magic here. (I have no idea what "normal stand-down" refers to.)
The question is really whether RealTek does a
better job at putting the processor to sleep while maintaining an association with the
access point.

### Leisure Power Save

_Leisure Power Save_ or Ameba LPS is the name RealTek gives to the RTL8710 mode where it maintains
an association with the access point while being asleep by periodically waking-up to catch a beacon.
LPS can be turned on easily and there is also a function to set the DTIM interval (the
equivalent to Espressif's listen-interval).

I decided to cut right to the chase and modified the RealTek sample SleepWithWifiIdle Arduino sketch
following the lines of [PVVX's
changes](https://github.com/pvvx/RtlDuino/tree/master/development/rtl87xx/libraries/PowerSave/examples/SleepWithWifiIdle)
to set the DTIM interval to every 4th beacon.
I added the code I used for the esp8266 to open a TCP connection to my local server and
send a message plus to toggle the LED so I could easily trigger the scope.

The resulting scope capture is shown below. It spans a total of 12 seconds and shows 3 loop
iteration wake-ups that are 5.4 seconds apart.

![RTL8710 Leisure Power Save](/img/low-power-wifi/rtl8710-leisure-power-save-all.png# nocaption)

Just in case you're wondering: nope, I didn't mix-up my scope captures, this is really an RTL8710
capture and not a misplaced esp8266 one! They really look that similar.

In the capture one can easily make out the various power levels and they
match up with the specs. The TCP connection happens very rapidly (in 11ms according to a packet trace
I captured) and one can see the very brief
wake-up periods every 400ms to catch a beacon.
Something I don't like at all are the very long
high-power periods at ~60mA: those will ruin the low-power performance!

Looking a bit closer at the activity, here is a zoom-in of the same capture focusing on one loop
iteration:

![RTL8710 Leisure Power Save zoomed](/img/low-power-wifi/rtl8710-leisure-power-save-zoom.png# nocaption)

The pulse on the yellow trace spans the entire loop iteration. The corresponding TCP packet
transmissions can be seen as power spikes. In addition, the capture shows a beacon with a
packet pending. The three TX power spikes of that wake-up period should look familiar:
power-save-off, then an ACK for the packet
received, and a power-save-on packet. The total duration is shorter than for the esp8266 or esp32:
the RTL8710 re-enters power save in just under 40ms.

Here is a Wifi packet capture of such a beacon with three packets to pick-up:

![RTL8710 Beacon Pick-up](/img/low-power-wifi/rtl8710-beacon-pickup.png# nocaption)

First comes the beacon with the TIM field stating that there's a packet for the RTL8710 (I checked
the packet details to confirm), then the RTL8710 sends a null packet without "P" flag, meaning it's
exiting power-save. The AP ACKs that packet and follows with a 3-packet sequence that it had stored
up. The RTL8710 ACKs each one and then 30ms after the last one sends a null packet with the "P" flag
to tell the AP that it is re-entering power-save mode.

Now to the very long periods at high power. The RTL8710 datasheet says the following about the
Leisure Power Save mode:

![RTL8710 LPS Spec](/img/low-power-wifi/rtl8710-lps-spec-1.png# w-80 ml-5 nocaption)
![RTL8710 LPS Spec](/img/low-power-wifi/rtl8710-lps-spec-2.png# w-80 ml-5 nocaption)

So somehow there is a 2-second timer involved and, looking at the scope captures, indeed one of the
high-power periods is a tad over 2 seconds long and the other a tad over 4 seconds. This pattern of
2-second and 4-second high-power periods repeats consistently although I can't quite figure out
what exactly triggers the periods.

In a last attempt at LPS I changed the settings to the ones I used with the esp32: a DTIM interval
of 50 (5 seconds) and a loop interval of 23 seconds. The following scope capture shows a total
elasped time of 1 minute:

![One minute of RTL8710 Leisure Power Save](/img/low-power-wifi/rtl8710-leisure-power-save-1m.png# nocaption)

In this trace the RTL8710 spends more time at RX and modem-sleep power levels 
than powering down to light-sleep. Clearly cranking the DTIM interval up does not help.

So far it does not look like the RTL8710 provides a better alternative to the Espressif
microcontrollers for low-power operation. The LPS mode may be usable given some tweaking but
I very much doubt that it can be made to use less power than the esp32.

The [next post](/2018/lp-wifi-rtl8710-deep) examines how the RTL8710 performs in deep-sleep with periodic
wake-up.
