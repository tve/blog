---
name: lp-wifi-esp32-1
title: Low-Power Wifi with the ESP32
date: 2018-11-26
thumbnail: "/img/low-power-wifi/esp32-set-up_sq.jpg"
project: low-power-wifi
categories:
- low-power
- wifi
- esp32
---

The esp32 is the successor of the esp8266 by Espressif. It is much faster, has two cores, more
memory, more I/O, more everything. Does it also have more low-power Wifi mojo?<!--more-->

### Experimental set-up

To test this I used an experimental set-up that is basically the same as with the esp8266.

![Experimental set-up](/img/low-power-wifi/esp32-set-up.jpg# fr w-40pct ml-3)

The set-up I use is as follows:

- an Adafruit Huzzah32 powered by a LiPo, modified to lower the deep-sleep current to 250uA,
- a jumper hacked into the Huzzah32's battery ground connector to measure the supply current,
- a uCurrent Gold to amplify the current (a 1 Ohm resistor would have probably done just fine given
  the magnitude of the currents), and
- an oscilloscope to measure the current and a probe on the LED gpio pin to trigger the scope.

The Wifi set-up is identical to the one used for the esp8266 tests.

### Deep-Sleep

The esp32 promises to complete each wake-cycle in less time than the esp8266 thanks to its faster
dual cores. Is that really the case or will it simply consume more power as it waits for Wifi responses?

Again, only experimentation provides an answer. The [test
sketch](https://github.com/tve/low-power-wifi/tree/master/esp32-deep-sleep) uses the Arduino framework and a
pre-release version of ESP-IDF v3.2 underneath. The code is adapted from the esp8266 version,
changing a few portions that were esp8266 specific. One notable difference is that the local port
number gets selected randomly automatically, so that's one less work-around in the app code.

The code has the same 4 modes as the esp8266 version.
They differ in the amount of information passed into the Wifi initialization functions
`WiFi.begin` and `WiFi.config`:

mode | ssid, passwd | channel, bssid | ip, mask, gw, dns
:---:|:---:|:---:|:---:
0 | ✔ |   |  
1 | ✔ | ✔ |  
2 | ✔ |   | ✔
3 | ✔ | ✔ | ✔

A quick look at the messages sent to the server shows the following:

```
Mode 0, Init 17 ms, Connect 798 ms, Total 815 ms, SSID test, IDF v3.2-dev-39-gaaf12390
Mode 1, Init 17 ms, Connect 205 ms, Total 222 ms, SSID test, IDF v3.2-dev-39-gaaf12390
Mode 2, Init 17 ms, Connect 766 ms, Total 783 ms, SSID test, IDF v3.2-dev-39-gaaf12390
Mode 3, Init 17 ms, Connect 163 ms, Total 180 ms, SSID test, IDF v3.2-dev-39-gaaf12390
```

We see the same behavior as with the esp8266 where modes that don't specify the channel/BSSID (0 & 2) take
significantly longer than modes that do (1 & 3). The times for the slow modes are less than half of
those for the esp8266, so that's good news. The not-so-good news is that the fast modes take
almost the same time.

### Open access point

Diving past the numbers in the table above, here is a scope capture of what mode 0 (specify only
SSID in `Wifi.begin()`) on an open access point looks like:

![ESP32 deep sleep scope capture](/img/low-power-wifi/esp32-deep-sleep-open-mode-0-all-annot.png)
_Scope capture of the esp32 waking from deep sleep and making a TCP connection to a server._

Given what we know from the esp8266 we can immediately assume that the long section (marked "probes")
corresponds to the esp32 probing all the channels. A packet dump confirms this assumption
and overall looks as one might expect, so I won't reproduce it here in its entirety.

Looking at packet dumps I kept stumbling over oddities during probing. One of them is that the esp32
does not reliably ACK probe responses from the AP even though all other packets I see are ACKed
reliably.
Here is a snippet that shows it ("->" refers to TX and "<-" to RX from the esp32 perspective):

```
              -> [Missed initial probe request]
  107 56.431s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  108 56.432s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  109 56.434s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  110 56.435s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  111 56.437s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  112 56.438s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  113 56.440s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  116 56.546s -> Probe Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 2 4.0 48.0 Mbit]
  117 56.551s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  118 56.553s -> Acknowledgment
  119 56.556s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  120 56.558s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  121 56.559s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  122 56.561s -> Acknowledgment
  125 56.667s -> Probe Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 2 4.0 48.0 Mbit]
  126 56.672s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  127 56.674s -> Acknowledgment
```

Specifically, packets 108 through 113 are retransmissions from the AP because the esp32 doesn't
respond with a Wifi ACK. Another oddity is that even after the esp32 ACKs a response (packets
117-118) it sends a fresh probe request later on (packet 125).

I puzzled over this for a long time and eventually reached the following hypothesis:

1. When the esp32 has to probe multiple channels it can only do so on one channel at a time, so it
   may miss whole sets of responses due to switching away to the next channel. (My captures are
   single-channel only.)
2. The probing tries to discover _all_ APs for the given SSID so it can pick the best one and
   therefore it goes through a certain number of request rounds and fixed wait periods
   even if some AP has already responded.

I'm sure there are additional nuances I'm not understanding, if you have more info I'd appreciate if
you could leave a comment!

The [next post](/2018/lp-wifi-esp32-2) optimizes away the probing and switches some security on.
