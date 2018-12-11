---
name: lp-wifi-esp-comparison
title: ESP8266 vs. ESP32 on Battery Power
date: 2018-12-06
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

![Deep-sleep - secure AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Secure AP 1.png# fr w-50pct nocaption)
![Deep-sleep - secure AP](/img/low-power-wifi/Deep-sleep periodic wake-up - Secure AP 2.png# w-50pct nocaption)

When using a secure access point the esp8266 and esp32 show roughty the same performance.
The reason is that while the esp32 uses more power when active it completes the WPA2-PSK
association more rapidly then the esp8266.

Adding MQTT and TLS to the exchange doesn't cost a lot of extra power because of the speed at which
the esp32 performs the crypto and the rapid Wifi packet exchanges.

### Light-sleep maintaining an association

The next charts use automatic light-sleep where the processors automatically go to sleep when there is no
activity and wake-up periodically to maintain an association with the access point.
The chart varies the iteration period and the number of background broadcast packets per second.

In the esp32 case the chart uses the values for the ["pushed"
version](/2018/lp-wifi-esp32-pushed#to-the-limit),
which sets the listen_interval
to 5 seconds and removes the ACK delay in LwIP.

The comparison is a little skewed because the esp8266 just sends a small amount of data over TCP
while the esp32 publishes a message over MQTT/TLS and receives a subscription message.
However, as we've seen before, the exact nature of the connection tends not to be the dominant
factor in the overall power consumption.

![Light-sleep - secure AP](/img/low-power-wifi/Light-sleep - Secure AP.png# w-100pct nocaption)

In light-sleep mode the esp32 performs much better than the esp8266 in large part due to the
esp8266's bug where it drops into modem-sleep instead of light-sleep about half the time after a
beacon.

The fact that the esp32 allows the listen interval to be set explicitly is also an advantage, not
just in terms of the numbers shown here but also because it reduces the impact of OOPS











### Conclusions






This series will be continued, stay tuned...
