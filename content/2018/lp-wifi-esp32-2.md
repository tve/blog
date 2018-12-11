---
name: lp-wifi-esp32-2
title: ESP32 Deep-Sleep with Periodic Wake-up
date: 2018-11-27
thumbnail: "/img/low-power-wifi/esp32-deep-sleep-secure-mode-3-all-sq.png"
project: low-power-wifi
tags:
- low-power
- wifi
- esp32
---

The long probing for the access point can be optimized away. In addition, it's not smart to use
open access points, so it's time to measure the cost of security!
<!--more-->

### Optimizing the probing

Mode 3 in
the [test sketch](https://github.com/tve/low-power-wifi/tree/master/esp32-deep-sleep)
specifies all possible Wifi parameters in `WiFi.begin()` and `WiFi.config()`
and therefore runs much faster:

![ESP32 deep sleep scope capture](/img/low-power-wifi/esp32-deep-sleep-open-mode-3-all-annot.png)
_Scope capture of the esp32 waking from deep sleep and making a TCP connection to a server._

The sketch changed a little bit for this scope capture and creates two shorter "blips" on the
yellow trace instead of a single long one:
one blip at the start of `setup()` until just before calling `Wifi.begin()` and a second starting when
`WiFi.status()` reports that it is connected (has an IP address) until just before calling
deep-sleep.

There are two things that immediately stand out on this trace. The first is that it takes longer for
the esp32 firmware to start-up (~270ms) than it takes for the sketch to fire-up Wifi and do its TCP
connection (~200ms)! I asked and was told by `@me-no-dev` that this is due to the extensive chip/memory
verification that occurs at boot time and that I wasn't missing some magic setting. The impact on
total power consumption is definitely noticeable but not quite as bad as it might seem at first
because the power consumption is much lower during this start-up period than during the active
portion.

The second noticeable aspect of this trace is that probing _still_ takes the majority of the time and
includes a long idle period. I have to admit that I just don't quite understand the intricacies of
probing... The overall packet trace is quite sweet and instructive:

```
  117 37.633s -> Probe Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 2 4.0 48.0 Mbit]
  119 37.637s <- Request who-has 192.168.0.130 tell 192.168.0.2, length 46
  120 37.638s -> Acknowledgment
  121 37.638s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 1 2.0 18.0 Mbit] CH: 6
  122 37.639s -> Acknowledgment
```

This was the probing! It took 121ms if we look at the time of the next packet in the dump below.
This matches the scope trace. The actual packets are a little odd: the first is the expected probe request
kicking things off, but then comes an ARP request from the server we're going to contact!
This is really the server trying to get a MAC address to send some
FIN packets from the previous sketch iteration. What's odd is that _the esp32 answers_ with
an ACK in packet 120. There is no ARP reply, though, which makes sense since it hasn't really
associated with the AP yet.
So somehow more than expected of the Wifi stack must be hooked-up even though there is no
association, I wonder whether there's an attack vector buried in here...

Quickly thereafter comes a probe response and ACK. What I don't understand, again, is why this is
followed by a ~110ms pause. It seems to me that the esp32 should not be waiting since it found the
AP specified in `WiFi.begin()`.
(I posted a [question](https://www.esp32.com/viewtopic.php?f=2&t=8202) on the Espressif Esp32 forum,
let's see...)

The next set of packets associate, send gratuitous ARPs, and locate the MAC address of the server.
All in all this takes 21ms:

```
  125 37.754s -> Authentication (Open System)-1: Successful
  126 37.755s <- Acknowledgment
  127 37.759s <- Authentication (Open System)-2:
  128 37.760s -> Acknowledgment
  129 37.761s -> Assoc Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 2 4.0 48.0 Mbit]
  130 37.762s <- Acknowledgment
  131 37.764s <- Assoc Response AID(1) :: Successful
  132 37.765s -> Acknowledgment
  133 37.767s -> Request who-has 192.168.0.130 tell 192.168.0.130, length 28
  134 37.768s <- Acknowledgment
  135 37.768s -> Request who-has 192.168.0.130 tell 192.168.0.130 , length 28
  136 37.769s <- Acknowledgment
  137 37.770s -> Request who-has 192.168.0.2 tell 192.168.0.130, length 28
  138 37.771s <- Acknowledgment
  139 37.773s -> Request who-has 192.168.0.130 tell 192.168.0.130, length 28
  140 37.774s -> Request who-has 192.168.0.2 tell 192.168.0.130, length 28
  141 37.774s <- Reply 192.168.0.2 is-at b8:97:5a:90:54:5a, length 46
  142 37.775s -> Acknowledgment
```

Now the TCP connection can occur:

```
  143 37.776s -> 192.168.0.130.50114 > 192.168.0.2.12345: Flags [S], seq 6509, win 5744, options [mss 1436], length 0
  144 37.777s <- Acknowledgment
              <- [Missed TCP SYN-ACK]
  145 37.782s -> Acknowledgment
  146 37.783s -> 192.168.0.130.50114 > 192.168.0.2.12345: Flags [.], ack 1120519907, win 5744, length 0
  147 37.784s <- Acknowledgment
  148 37.785s -> 192.168.0.130.50114 > 192.168.0.2.12345: Flags [P.], seq 0:87, ack 1, win 5744, length 87
  149 37.786s <- Acknowledgment
  150 37.787s -> 192.168.0.130.50114 > 192.168.0.2.12345: Flags [F.], seq 87, ack 1, win 5744, length 0
  151 37.787s <- Acknowledgment
              <- [Missed TCP ACK]
  152 37.791s -> Acknowledgment
              <- [Missed TCP FIN]
  153 37.791s -> Acknowledgment
  154 37.792s -> 192.168.0.130.50114 > 192.168.0.2.12345: Flags [.], ack 2, win 5743, length 0
  155 37.793s <- Acknowledgment
```

And it's a wrap! Other than the pause after the probe response and the two rounds of ARP requests
because my linux server is lazy in responding all this looks about as good as one might expect!

### Secure access point

Does security run into the same power increase as it does with the esp8266?
The following scope trace answers this question:

![ESP32 deep sleep scope capture](/img/low-power-wifi/esp32-deep-sleep-secure-mode-3-all-annot.png)
_Scope capture of the esp32 waking from deep sleep and making a TCP connection to a server on
a secure (WPA2) Wifi network._

Yes, we still have to pay a price when we choose security!
(Note the 200ms/div here vs. 50ms/div in the open-AP scope capture further up.)
The primary difference with respect to the open
access point trace is the approx 700ms period of elevated power consumption between the end of the
probes and the association (I verified this with a packet dump).
I can only surmise that this is some crypto calculation but I spent quite some time
looking through the WPA2-PSK standard and all the crypto key exchange happens way later.
Unless there is something that can be pre-computed or perhaps some random number
initialization I don't quite grasp what is going on.
I posted another [question](https://www.esp32.com/viewtopic.php?f=2&t=8203) on the Espressif Esp32 forum,
let's see whether anyone can provide an insight.

### Power calculations

The following table and chart show the expected run-times on a 1000mAh battery assuming wake-up intervals
from once per hour down to once every 20 seconds. The "open" columns refer to open access points and
"secure" to WPA2-PSK. The "act%" columns refer to the percent of total power consumption that comes
from the active period (startup+wake).

sleep [min] | open [mA] | open [days] | open [act%] | secure [mA] | secure [days] | secure [act%]
---:| ---:| ---:| ---:| ---:| ---:| ---:|
60 | 0.02 | 2040.8 | 51% | 0.04 | 952.4 | 77%
10 | 0.07 | 574.7 | 86% | 0.21 | 196.1 | 95%
1 | 0.64 | 65.6 | 98% | 2.04 | 20.5 | 100%
0.33 | 1.90 | 21.9 | 99% | 6.15 | 6.8 | 100%

![ESP32 Deep-Sleep with Periodic Wake-up](/img/low-power-wifi/ESP32 Deep-Sleep with Periodic Wake-up.png# fr w-60pct nocaption)

The assumptions underlying the table and chart are:

metric | value
--- | ---
startup [mA] | 50
running [mA] | 120
sleeping [mA] | 0.01
startup [ms] | 270
open wake [ms] | 200
secure wake [ms] | 900

The following scope capture adds a purple trace showing the power consumption integral.
This shows that the majority of the power is spent when the esp32 is running actively at 125-150mA.
The 270ms at start-up don't add a significant amount to the power consumption, but they do add a
potentially problematic delay if the wake-up is caused by some external event (i.e. gpio input) and
the esp32 needs to react quickly to it.

![ESP32 deep sleep scope capture](/img/low-power-wifi/esp32-deep-sleep-secure-mode-3-all-power.png)
_Scope capture of wake-up with secure AP with power consumption integral added in purple._

Going back to the original goal of achieving a run-time of a week to a month on a 1000mAh battery
the esp32 provides a number of options when using the deep-sleep mode. Specifically, it can wake up
every 20 seconds when using an open access point (not really recommended!) and every minute when
using a secure access point.

The [next post](/2018/lp-wifi-esp32-mqtt) goes a step further and explores what happens if the esp32
makes a TLS connection to an MQTT server and sends & receives some messages.

### Addendum

The esp32 provides an additional explicit light-sleep mode that has not been
explored here and that is closely related to deep-sleep.
Explicit light-sleep puts the cpu and radio into light-sleep mode for a fixed duration just
like deep-sleep does, but it preserves memory content. This allows the periodic wake-up to be
implemented without loosing memory state but at the cost of higher power consumption. The Wifi has
to be turned off prior to entering light-sleep, so the wake-cycle power is expected to be very similar
to the deep-sleep case since most of the power is spent on connecting to Wifi (especially in the
secure network case).
