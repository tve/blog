---
name: lp-wifi-esp-comparison
title: ESP8266 vs. ESP32 on Battery Power
date: 2018-12-10
thumbnail: "/img/low-power-wifi/esp8266-esp32.jpg"
project: low-power-wifi
tags:
- low-power
- wifi
- esp32
- esp8266
---

Having examined the esp8266 and esp32 power consumption in detail it is time to
compare the two.
<!--more-->


In all the comparisons in this post the following definitions/assumptions are made:

- run-time is calculated based on an idealized 1000mAh battery,
- the access point used is fairly idle, but there are other hosts on the network and thus 
  there is a constant low rate of background broadcast packets (ARP requests, spanning tree, etc),
- TCP connections contain a minimal data packet,
- TLS connections use a PSK cipher suite, and
- MQTTS uses TLS and contains a publish message and a subscription message delivery.


### Deep-sleep with periodic wake-up

The comparison starts with the deep-sleep with periodic wake-up mode of operation.
It is assumed that all the optimizations described in earlier posts are made,
such as providing all the parameters to `wifi.begin` and `wifi.config`.

#### Open access point

The two following charts compare the run-time in days between the esp8266 and the esp32
when running a test app that sends a simple packet to a server over TCP. The left
chart shows wake-up intervals of 10 and 60 minutes while the right-hand one shows wake-up intervals
of 20 seconds and 1 minute. This could really be one chart except that a single vertical scale would
cause the 20-second interval bars to be one pixel tall...

![Deep-sleep - open AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Open AP 2.png# fr w-50pct nocaption)
![Deep-sleep - open AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Open AP 1.png# w-50pct nocaption)

The charts show that when using an open access point the esp8266 uses less power than the esp32.
The reason is that the consumption is dominated by the current used when active,
which is higher in the esp32 case,
and both are active for about the same amount of time,
which is dominated by the probing for the access point.

#### Secure access point

The next two charts compare the run-times when using a secure AP and adding MQTTS/TLS in the esp32
case.

![Deep-sleep - secure AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Secure AP 2.png# fr w-50pct nocaption)
![Deep-sleep - secure AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Secure AP 1.png# w-50pct nocaption)

When using a secure access point the esp8266 and esp32 show roughly the same performance.
The reason is that while the esp32 uses more power when active it completes the WPA2-PSK
association more rapidly then the esp8266.

Adding MQTT and TLS to the exchange doesn't cost a lot of extra power because of the speed at which
the esp32 performs the crypto and the rapid Wifi packet exchanges.

### Light-sleep maintaining an association

The next chart shows automatic light-sleep where the microcontroller automatically goes to sleep
when there is no activity and wakes-up periodically to maintain an association with the access point.
The chart varies the iteration period and assumes one broadcast packet causing a wake-up every 10
seconds.

![Light-sleep - secure AP](/img/low-power-wifi/Light-sleep - Secure AP.png# fr w-50pct ml3 nocaption)

In the esp32 case the chart uses the values for the ["pushed"
version](/2018/lp-wifi-esp32-pushed#to-the-limit),
which sets the listen_interval
to 5 seconds and removes the ACK delay in LwIP.

The comparison is a little skewed because the esp8266 just sends a small amount of data over TCP
while the esp32 publishes a message over MQTT/TLS and receives a subscription message.
However, as we've seen before, the exact nature of the connection tends not to be the dominant
factor in the overall power consumption.

In light-sleep mode the esp32 performs much better than the esp8266 in large part due to the
esp8266's bug where it drops into modem-sleep instead of light-sleep about half the time after a
beacon.

The fact that the esp32 allows the listen interval to be set explicitly is also an advantage, not
just in terms of the numbers shown here but also because it reduces the impact of Wifi broadcast
activity.

### Conclusions

To conclude the esp8266 and esp32 low-power wifi exploration the graph below summarizes the
tradeoffs between the two microcontrollers and the two sleep modes. It's a bit of an eye chart with
so many lines but the lines comes in pairs making it not all that difficult to walk though (see
below). The chart
shows the run-time on an idealized 1000mAh battery on the vertical axis (i.e. higher is better) vs.
the interval at which the code is waking up to communicate.
All the numbers are based on modeling the observed behavior of the microcontrollers.

![Esp8266 vs. esp32, light-sleep and deep-sleep](/img/low-power-wifi/ESP8266 vs. ESP32 in Light-Sleep and Deep-Sleep.png# w-100pct nocaption)

The lines that go off the top of the chart are for deep-sleep with periodic wake-up using an
insecure open access point.
Deep-sleep without crypto uses the least power of all options when
iteration times exceed a few seconds.
The esp8266 fares better than the esp32 because it uses less power when active and has a shorter
start-up sequence. Both cases specify all parameters to `WiFi.begin` and `WiFi.config`, i.e. they're
the optimized cases.

The other two straight lines that are right on top of one another are for deep-sleep with periodic
wake-up but using a secure WPA2-PSK access point.
Somewhat surprisingly the two microcontrollers perform basically the same, although the comparison
is slightly skewed because the esp32 communicates with an MQTT/TLS server at each wake-up while the
esp8266 only sends a brief TCP message.
The esp32 performs the WPA2 crypto faster than the esp8266 but uses more power while doing so.

The two solid curved lines are for automatic light-sleep where `loop()` iterates periodically and
there is approximately one broadcast packet every 10 seconds on the network that also wakes-up the
microcontroller. In both cases a secure access point is used.
The lines are curved because at short intervals the `loop()` wake-up is the dominant effect while at
long intervals the broadcast packet wake-up becomes dominant.
Here the esp32 performs much better than the esp8266 primarily because it reliably goes back to light-sleep.
In addition, its power save listen interval is 5x as long and
it goes to sleep more rapidly after waking up.

The final two dotted lines are idealized: they cannot be achieved with the current Espressif WiFi
firmware.
In the esp32 case the dotted line represents automatic light-sleep where the lingering interval
before re-entering power-save mode has been shortened from 50ms to 10ms. In the esp8266 case it
represents fixing a bug that puts the microcontroller into modem-sleep instead of light-sleep about
half the time. These two lines show what might be achievable if Espressif fixed these issues.

Comparing the esp32 light-sleep and deep-sleep cases shows a cross-over around an interval of 60
seconds. This means that use-cases that only require a wake-up every minute or longer should
use deep-sleep as expected, but use-cases where the microcontroller needs to remain reachable at all
times work better in the light-sleep mode.

My personal opinion on this comparison is biased towards security, and that strongly favors the
esp32 not only because of the power consumption but also because of its larger memory, which makes
the use of TLS much more practical.

While there are some more loose end I'd like to explore with the esp8266 and esp32 the
[next post](/2018/lp-wifi-rtl8710-lps) switches gears a little and investigates the RealTek RTL8710
in the form of the RTLDuino.
