---
title: RTL8710 Deep-Sleep with Periodic Wake-up
date: 2018-12-15
thumbnail: "/img/low-power-wifi/rtl8710-deep-sleep-open.png"
project: low-power-wifi
tags:
- low-power
- wifi
- rtl8710
---

Is the RTL8710 faster at waking up and connecting to a Wifi network than the Espressif
microcontrollers?

### Open Network

The sketch used for this test is very simple: connect to the open Wifi network, connect to a local server
using TCP and send a short message, then go back to sleep.

Unfortunately, the way LwIP is compiled into the firmware exposes the same local port issue
as in the esp8266, e.g., LwIP always uses the same port for the first connection and this causes
connection resets and additional delays.
I had to modify the Arduino WiFiClient library to add a hack so it explicitly binds the socket to a
pseudo-random local port.
After this fix the result is the following scope capture:

![RTL8710 Deep-Sleep Scope Capture using an Open Network](/img/low-power-wifi/rtl8710-deep-sleep-open.png#)

The wake cycle starts with a 270ms initialization phase before `setup()` is called. Then, after
another 250-300ms of RF init or calibration the rtl8710
spends 1 second probing for the access point. There are 11 spikes during this time which matches the
11 Wifi channels in the US, so evidently it spends about 90ms per channel. Then comes a 300ms dead period
where nothing happens, not unlike in the Espressif microcontrollers, and finally the rtl8710
associates and gets an IP address via DHCP all in about 100ms.

At the end of the DHCP exchange the rtl8710 sends a gratuitous ARP to detect whether anyone else has
that same IP address. This repeats twice over about 500ms before the firmware reports to the sketch
that wifi is connected. The sketch then opens the TCP connection, which results in an ARP request
for the destination server followed by the TCP connection in almost no time.

Overall, the performance is roughly in-line with the equivalent unoptimized sketch on the esp8266.
I then tried to perform the same optimizations but the Arduino libraries provided with the
rtl8710 allow neither the BSSID nor the channel number to be specified. It turns out, however, that
the underlying Wifi firmware does allow the BSSID to be set (but not the channel),
so I hacked in calls to do that.
Sadly I saw absolutely no difference in the probing.
The end result is that the wake-cycle on an open network takes about ten times as long as on the
esp8266 or esp32.

### Secure Network

The next scope capture shows the same sketch with a secure network. 
It uses the same simple TCP connection instead of the MQTT/TLS pub/sub used in the esp8266 and esp32
sketches because the TLS library does not support any of the PSK ciphers.

![RTL8710 Deep-Sleep Scope Capture using a Secure Network](/img/low-power-wifi/rtl8710-deep-sleep-secure.png#)

Here the horizontal scale had to be changed from 200ms/div to 500ms/div. The probing on the left
still takes about 1 second and once the rtl8710 decides to associate it gets through the remainder
of the sketch in about 600ms like in the open network case. But in the middle there is a 3 second
pause where nothing seems to happen, very similar to the delay in the esp8266 and esp32, but longer.

The bottom line here is that the wake-cycle takes a little over 5 seconds, which is more than twice
as long as for the esp8266 and esp32.

### Conclusions

It is pretty clear that for low-power Wifi the RTL8710 cannot compete with the esp32: it's "leisure
power sleep" uses much more power than the esp32's automatic light-sleep and the deep-sleep with
periodic wake-up it takes much longer to associate with an AP. It also cannot compete with the esp8266
in deep-sleep mode but is roughly comparable in light-sleep mode where both have bugs/mis-features
that keep them awake half the time when they should be sleeping.

I did revisit the summary chart from the esp8266 vs. esp32 comparison and added the rtl8710 using
bold lines. The actual leisure sleep performance assumes a 50/50 split between light-sleep
and periods where it stays powered-up due to the "N packets in 2 seconds" stay-awake algorithm in
the Wifi firmware. I'm not sure what the correct split really is as I've seen some variability
depending on the number of background packets, the loop iteration interval, and other factors that I
don't understand.

![RTL8710 added to the esp8266 vs esp32 comparison](/img/low-power-wifi/ESP8266 vs. ESP32 vs. RTL8710 in Light-Sleep and Deep-Sleep.png# w-100pct nocaption)

I did model what would happen if RealTek fixed the leisure sleep issue and the performance would
actually lie somewhere between the esp8266 and the esp32, which would be pretty sweet. The
deep-sleep with periodic wake-up performance is worse than for the esp32 and esp8266 due to the long
time it takes to connect with the access point.

The [next post](/2018/lp-wifi-other) which will take a brief look at some other low-power Wifi microcontrollers.
