---
name: lp-wifi-esp32-dfs
title: ESP32 Dynamic Frequency Scaling
date: 2018-12-04
thumbnail: "/img/low-power-wifi/esp32-light-sleep-sq.png"
project: low-power-wifi
categories:
- low-power
- wifi
- esp32
---

Dynamic frequency scaling allows the esp32 to go into light-sleep mode when it is idle.<!--more-->

In the ESP IDF framework dynamic frequency scaling with light-sleep is enabled by turning on power
management (a compile-time option) and then calling a few functions as described in this
[documentation](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/power_management.html).
Unfortunately the arduino framework does not currently support enabling the power management because
its interaction with peripheral clocks is non-trivial.

To evaluate the potential of the frequency scaling and light sleep I started with the `power_save`
[ESP IDF example app](https://github.com/espressif/esp-idf/tree/master/examples/wifi/power_save).
At first I had difficulties getting it to work, which resulted in a 
[ticket](https://github.com/espressif/esp-idf/issues/2711) and
a quick [patch](https://github.com/espressif/esp-idf/issues/2711#issuecomment-441863113).
This made the example app work on my Wifi network.

I then modified the example app slightly and set the listen interval to 9, i.e., wake up every 900ms
to check the beacon. The app just connects to Wifi and then does nothing, which is not terribly
representative, so I generated some Wifi activity by running a ping to the esp32 with an interval
of 6 seconds.
A snippet of the ping output is:

```
64 bytes from 192.168.0.165: icmp_seq=60 ttl=255 time=79.9 ms
64 bytes from 192.168.0.165: icmp_seq=61 ttl=255 time=528 ms
64 bytes from 192.168.0.165: icmp_seq=62 ttl=255 time=54.3 ms
64 bytes from 192.168.0.165: icmp_seq=63 ttl=255 time=504 ms
64 bytes from 192.168.0.165: icmp_seq=64 ttl=255 time=24.8 ms
64 bytes from 192.168.0.165: icmp_seq=65 ttl=255 time=471 ms
64 bytes from 192.168.0.165: icmp_seq=66 ttl=255 time=2.33 ms
64 bytes from 192.168.0.165: icmp_seq=67 ttl=255 time=440 ms
64 bytes from 192.168.0.165: icmp_seq=68 ttl=255 time=886 ms
64 bytes from 192.168.0.165: icmp_seq=69 ttl=255 time=411 ms
```

The response time varies as expected from anywhere between a couple of milliseconds up to almost
900ms, which is the interval at which the esp32 wakes up.

A scope capture of the power consumption looks as follows:

![Automatic light sleep](/img/low-power-wifi/esp32-light-sleep-all.png)
_Esp32 in automatic light sleep with a listen interval of 900ms and being pinged every 6 seconds._

The capture uses a 1 second per horizontal division and thus shows 12 seconds total. Two pings can
be discerned from the TX power spikes that are 6s apart. An additional power spike must be due to
some broadcast traffic on the network that the esp32 picks up. The wake-up every 900ms to listen to
the beacon can also be seen and finally the most frequent power level is very close to zero in light
sleep.

The following zoomed scope capture magnifies the period where some broadcast traffic and a ping are
received. One can now see the familiar pattern where the esp32 wakes up, transmits a power-save-off
packet, receives packets from the AP, potentially responds (in the case of the ping), then lingers
for 50ms and finally sends a power-save-on packet before dropping to sleep.

The capture also shows some timer activity every 100ms and the light-sleep power save level. Not
really discernible in this capture is the fact that the light-sleep consumption is about 1.1mA.

![Automatic light sleep zoomed](/img/low-power-wifi/esp32-light-sleep-zoom.png)

The big question now is what the average power consumption is and as previously mentioned in the
esp8266 light-sleep post this really depends on the amount of network activity that causes the esp32
to wake up! I measured the following average power consumption figures for this particular set-up:

- overall with pings and bcast traffic: 6.1mA
- 100ms period without wifi activity and one "timer tick": 2.1mA
- 100ms period with wifi RX-only wake-up for beacon: 7.1mA
- 100ms period with wifi wake-up incl packet pick-up: I forgot to measure, but since at least 50ms
  are spent lingering at ~115mA the average is certainly above 50mA

It is quite clear that the power consumption is primarily related to how frequently the esp32
wakes-up. I don't understand why Espressif doesn't:

- allow the listen interval to be set larger than 10
- allow the lingering period to be controlled by the app

Obviously tweaking these settings has an effect on how responsive the esp32 is to network traffic
and requests, but surely the app designer knows better what is appropriate in a given setting.

The [next post](/2018/lp-wifi-esp32-pushed) pushes the limits of automatic light sleep and calculates
run-times.
