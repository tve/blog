---
name: lp-wifi-esp8266-assoc
title: ESP8266 Maintaining an Association
date: 2018-11-25
thumbnail: "/img/low-power-wifi/esp8266-light-sleep-open-sq.png"
categories:
- low-power-wifi
- low-power
- wifi
- esp8266
---

In automatic light-sleep mode the esp8266 goes to sleep when there is no activity. From there it can
wake-up due to a timer or due to Wifi activity.
<!--more-->

Used on its own, light-sleep mode wakes up according to the access point's DTIM setting, i.e., the
period at which beacons contain a map of pending packets. In a recent SDK Espressif added a
twist on this, which enables the application to specify the period explicitly, in which case the access
point's setting is ignored. This allows the esp8266 to control its wake-up interval instead of being
at the mercy of the settings in the access point.

The assumption behind the explicit interval setting is that the access point most likely keeps
pending packets around for a while before discarding them. In theory, it only has to keep them for a
DTIM period but when the network is congested that's most likely too short anyway.

The [test sketch](https://github.com/tve/low-power-wifi/tree/master/esp8266-light-sleep)
starts by connecting to the access point and putting the esp8266 into light-sleep mode with an
interval of 5 beacons, i.e., 500ms. The `loop()` then consists of setting a gpio pin to high to
trigger the scope, opening a TCP connection to a test server, sending some data, and closing the
connection. It then sets the gpio pin to low and calls `delay(2000)`. At that point there's nothing
more for the esp8266 to do, so we expect it to go into light sleep mode for 2 seconds, interrupted
only by the periodic wake-up every 500ms to catch a beacon.

So long for the theory, the following screen capture shows what happens in practice. This is using
an open access point for starters.

![Light sleep](/img/low-power-wifi/esp8266-light-sleep-open-all.png)
_Scope capture of 12 seconds in automatic light sleep mode with a `loop()` iteration in the middle._

The yellow trace shows the gpio pin which briefly goes high in the center of the capture (not really
visible). Each horizontal division corresponds to one second, the full capture lasts 12 seconds. The
blue trace shows power consumption at 20mA per vertical division.

In the blue trace there are power spikes up to about 75mA clearly visible every 500ms.
These correspond to the wake-ups where the esp8266 catches a beacon.
Some of these spikes last longer (see green arrows) and contain some faint TX power spikes.
These correspond to periods where the beacon contains a flag that there's
a packet pending and then the esp8266 asks for the packet to be delivered.
In the middle of the trace more transmission activity can be seen, which is when the `loop()`
iteration occurs. It is also noticeable by the fact that it's offset with respect to the regular
500ms wake-ups.

So far everything looks as expected, but the power consumption when in sleep mode is not as
expected! In this particular capture, half the time when the esp8266 powers down after a beacon it
goes into light-sleep mode consuming less than 2mA. The the other half of the time it goes into
modem-sleep, which consumes about 16mA, ouch! Other captures show a different pattern and sometimes
more than half the time is spent in modem-sleep.
As we will see, this frequent operation at 16mA really eats into the run-time!

The following screen capture zooms in on the activity in the middle of the previous capture where
the `loop()` iteration and corresponding TCP connection occurs.

![Light sleep magnified](/img/low-power-wifi/esp8266-light-sleep-open-zoom.png)
_Scope capture of a `loop()` iteration._

Towards the left the blue power trace shows a wake-up for a beacon. It lasts about 15ms and
evidently there was no packet to retrieve. Afterwards the esp8266 goes into modem-sleep until the
loop delay expires. As it wakes up the yellow trace shows the `loop()` iteration lasting roughly
30ms and the blue trace shows a lot of Wifi activity, as we would expect. But when the sketch goes
back to sleep, the esp8266 does not power down, instead it lingers in high power mode for about
80ms, transmits a packet and goes to sleep (this time into light-sleep).

Here is another capture, this time zooming in on a beacon where the access point had a packet to
deliver to the esp8266:

![Light sleep magnified](/img/low-power-wifi/esp8266-light-sleep-open-beacon.png)
_Scope capture of a beacon with data to be delivered._

Somehow there are three transmissions involved in receiving the pending packet.
The best way to understand what is happening is to look at a packet capture:

```
  125   0.304s <- Beacon (test)
  126   0.307s -> Null data packet
  127   0.307s <- Acknowledgment
               <- [Missing ICMP request packet]
  128   0.314s -> Acknowledgment
  129   0.316s <- 192.168.0.106 > 192.168.0.2: ICMP echo reply
  130   0.316s -> Acknowledgment
  132   0.396s -> Null data packet
  133   0.396s <- Acknowledgment
  134   0.407s <- Beacon (test)
```

The first packet is a beacon that presumably contains a flag telling the esp8266 that there is a
packet pending. 3ms later the esp8266 requests packet delivery by sending an empty data packet with
a header flag set to indicate that it is exiting power save mode, i.e., it is now receiving
continuously, which means that the AP can send stored packets and any additional incoming ones at will.

After the Wifi-level ACK of the power-save-end packet the AP sends the data packet, which is an ICMP
ping request. Unfortunately the monitoring adapter missed it but one can see the ACK from the
esp8266 immeditaely after. Just 2ms later the esp8266 responds with an ICMP reply packet, which gets
ACKed. At this point the esp8266 doesn't know whether the AP has additional packets stored for it,
so it has to wait for "some time" to give the AP a chance to transmit them. From the timing column
it is evident that 80ms go by until the esp8266 sends another packet (number 132), which is another
empty data packet but this time with a power-save start flag telling the AP to buffer up any
packets. The last two packets are the ACK for the empty data packet and then some time later the
next beacon comes in.

The 80ms of delay are very consistent and can be seen both in this packet capture as well as the
scope capture above (the two don't show the exact same events).

I also looked at the performance when using a secure access point and surprisingly there is no
change in behavior to be seen. The TCP packet exchange may be happening a tad slower but when
looking at two traces side-by-side it is impossible to determine which is from an open vs. secure
access point.

### Putting it all together

From all evidence the esp8266's automatic light sleep mode operates more or less as expected, but
there is at least one bug and one unexpected issue:

- setting a light sleep interval causes the esp8266 to indeed go to sleep for the requested period
  between brief wake cycles to catch a beacon
- there is a clear bug in that after a wake-cycle the esp8266 does not systematically go to
  light-sleep mode and instead only goes into modem-sleep where it consumes 10 times more power
- when the esp8266 is stuck in modem-sleep it stays there until the next wake-up for a beacon, this
  means that the light-sleep interval setting is practically useless
- on the otherwise idle test Wifi network it takes very little time to wake-up, catch a beacon,
  request packet delivery and receive the packet
- after receiving the pending packet(s), there is a 80ms extra penalty where the esp8266 lingers
  with the receiver on, consuming around 70mA, before it goes back to sleep

A major issue with the lingering is that it is incurred each time some packet arrives, which is not
under the control of the esp8266 or its application. In the example above I used an ICMP ping so I
could relatively easily capture the activity, but under normal circumstances if nothing wants to
talk to the esp8266 there is still a regular cadence of broadcast packets that come through and wake
the esp8266 up.

A perhaps extreme test is to ping the esp8266 every second. In that case it never goes to sleep at
all, keeping the receiver going at 70mA full-time. What this highlights is that the esp8266 is
really at the mercy of the network environment and thus its battery run-time can be compromised by
completely unrelated activity.

The following table shows the average power consumption and run-time on a 1000mAh battery in a
variety of scenarios. The _iter period_ column specifies the rate at which `loop()` iterations run,
the _pkts per second_ specifies the number of beacon-induced wake-up periods there are per second,
the _ideal_ columns show the resulting average current and run-time if the esp8266 always properly
went to light-sleep and the _actual_ columns show a 50-50 mix of light-sleep and modem-sleep.

iter period [sec] | pkts per second | ideal [mA] | ideal [days] | actual [mA] | actual [days]
---:| ---:|---:|---:|---:|---:
1 | 0.1 | 12.6 | 3.3 | 19.8 | 2.1
10 | 0.1 | 4.4 | 9.5 | 11.6 | 3.6
60 | 0.1 | 3.6 | 11.5 | 10.9 | 3.8
1 | 1 | 17.6 | 2.4 | 24.9 | 1.7
10 | 1 | 9.4 | 4.4 | 16.7 | 2.5

The assumed durations and consumption levels are:

Light sleep modes | duration [ms] | current [mA]
--- | ---:| ---:
light-sleep | 500 | 1.5
modem-sleep | 500 | 16
beacon check | 10 | 70
lingering | 80 | 70
loop iteration | 50 | 70

It is quite clear from the table that the light-sleep mode is significantly more power-hungry than
the deep-sleep modes explored in a previous blog post.
In order to get a run-time of more than a couple of days one has to somehow ensure that there are
only very few broadcast packets on the network.

To get a better sense of where the power is going
the following table breaks down the % of power consumed in each mode for two of the rows above:

iter period [sec] | pkts per second | ideal [mA] | sleep [%pow] | beacon [%pow] | linger [%pow] | loop [%pow]
---:| ---:|---:|---:|---:|---: | ---:
10 | 0.1 | 4.4 | 34% | 32% | 26% | 8%
10 | 1 | 9.4 | 16% | 15% | 65% | 4%

The first row corresponds to contacting a server every 10 seconds and receiving some broadcast
packet every 10 seconds as well.
The run-time, if light-sleep was operating properly, would be 9.5
days, which meets the "run for at least a week" goal.
Given the even distribution of power consumption there is not one thing one could fix to
reduce the power significantly.
The second row shows the same server-contact rate but a 10x higher broadcast rate.
In that operating regime the power consumed by lingering after a packet becomes the single largest
power hog.

### Conclusion

This concludes the analysis of the esp8266's sleep modes.
It is pretty clear that keeping the esp8266 associated to an access point cannot be done with
run-times of more than about a week, unless one uses a sizeable battery. In addition, the actual
run-time is difficult to predict and may vary significantly over time because a lot depends on other
activity on the Wifi network.

To achieve long run-times one has to use deep-sleep and wake-up only intermittently. In that case
the run-time can be predicted with reasonable accuracy, although the time it takes the esp8266 to
successfully associate with the AP at each wake-up may vary quite a bit.

An unfortunate reality is that when using deep-sleep the power consumption and thus run-time
varies dramatically between open Wifi networks and secure networks.
Specifically, the penalty for security can be a factor of 10x!
In contrast, the light-sleep mode shows little difference between open and secure networks.

The [next post](/lp-wifi-esp32-1) examines whether the Espressif Esp32 is a better choice for
low-power wifi operation.

[Low-power Wifi series index](/categories/low-power-wifi)
