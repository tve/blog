---
name: lp-wifi-esp32-pushed
title: ESP32 Pushing the Envelope
date: 2018-12-09
thumbnail: "/img/low-power-wifi/max.png"
project: low-power-wifi
tags:
- low-power
- wifi
- esp32
---

With a few more hacks the esp32 light-sleep power consumption can be pushed to the limit. This post
explores the limit and calculates light-sleep mode run-times.
<!--more-->

### To the limit

The [previous post](/2018/lp-wifi-esp32-light) uncovered a number of problems with the automatic light
sleep mode.
In addition, the Arduino framework does not officially support the esp32's automatic power management,
and hence I used a trivial app to measure just the basic behavior.
In order to push the envelope I hacked ESP-IDF as well as the arduino framework.
I configured ESP-IDF to include the arduino framework
as a "component". The steps for that are best outlined in [this reply to a
ticket](https://github.com/espressif/arduino-esp32/issues/1142#issuecomment-367867856).

Using this method it is possible to make a number of fixes to push the envelope and measure real
application-level communication -- an encrypted MQTT exchange in my case.
The specific changes are:

- remove the ACK delay in LwIP (this will have negative effects on high bandwidth connections)
- enable dynamic power management and frequency scaling while using arduino
- push the listen_interval (how long to sleep between beacon-check wake-ups) to large values
- run the MQTT/TLS sketch used in previous posts

The specific settings I experimented with are:

- listen_interval = 5 seconds (e.g. a value of 50)
- wake-up interval to pub/sub MQTT messages = 23 seconds (I picked an odd interval)
- MQTT keep-alive = 70 seconds (effectively no keep-alives are sent)

Another setting that could be tweaked is to disable the periodic sending of gratuitous ARPs by LwIP,
but because it only happens once a minute I'm not convinced it makes a real difference here. 
The setting I really wanted to change as well is the Wifi RX lingering period before re-entering power-save
mode, but it appears to be buried in the Wifi binary blob.

After making these changes I took a fair number of measurements and had some difficulties interpreting
them because the loop iterations are not always successful. I was using my general home wifi and it
appeared as if the esp32 had difficulties maintaining the association. I switched to a test AP (on
the same LAN) and the association appeared more stable. I suspect that a listen interval of 5
seconds is too long but I left it as-is in order to get a "very best case" number. For a real
deployment one most likely has to tweak this setting to find a happy medium.

![Esp32 auto power save scope capture over 60 seconds](/img/low-power-wifi/esp32-auto-light-sleep-scope.png)
_Scope capture showing the esp32 automatic light sleep over a period of 60 seconds with two
wake-ups to exchange MQTT packets over TLS and several additional wake-ups to pick-up broadcast
packets. The average power consumption is approx 3.8mA and not 14.9mA as incorrectly averaged by the
scope._

The scope capture above shows the general behavior with two loop iterations just barely visible in
the yellow trace. The corresponding MQTT exchanges can be seen in the blue trace that captures the
power consumption. The Wifi wake-ups every 5 seconds are clearly visible and a number of them result
in the transmitter powering up to pick-up a packet that is pending at the AP. Unfortunately the
average power consumption displayed by the scope is inaccurate due to the degree of averaging being
performed in this capture of 60 second duration.

In order to get an accurate average power consumption reading I used a 1F supercap, which I measured
to have an actual 940mF (the tolerance spec is -20%/+80%).
I ran the app for 10 minutes and measured the voltage
drop which resulted in a calculated average power consumption of 3.8mA.

### Esp32 run-time in automatic light-sleep

With all the measurements we can put together a model for the power consumption when using automatic
light-sleep. The following chart shows the power consumption in terms of the expected
run-time on a 1000mAh battery as the wake-up interval is varied from 1 second to 60 seconds.

![ESP32 automatic light-sleep run-times](/img/low-power-wifi/esp32-auto-light-sleep-chart.png#
nocaption w-90pct center)

Three different models are shown:

- the _std_ model uses the off-the-shelf power management features as available in the ESP-IDF,
- the _pushed_ model uses the hacks described in the previous section, and
- the _ideal_ model shows what could hypothetically be achieved by reducing the lingering time to 10ms.
- the _deep-sleep_ model uses deep-sleep with periodic wake-up as [previously described](/2018/lp-wifi-esp32-2)

All the models assume:

<table><tbody>
<tr><th>operation</th><th>std model</th><th>pushed model</th><th>ideal model</th></tr>
<tr><td>light-sleep current</td><td colspan="3" align="center">1.1mA</td></tr>
<tr><td>listen_interval</td><td align="center">1s</td><td colspan="2" align="center">5s</td></tr>
<tr><td>beacon check</td><td colspan="3" align="center">10ms @125mA</td></tr>
<tr><td>lingering</td><td colspan="2" align="center">50ms @125mA</td><td align="center">10ms @125mA</td></tr> 
<tr><td>loop iteration</td><td colspan="3" align="center">20ms @125mA</td></tr>
<tr><td>broadcast packet rate</td><td align="center">1 pps and 0.1 pps</td><td colspan="2" align="center">0.1 pps</td></tr>
<tr><td>security</td><td colspan="3" align="center">WPA2-PSK &amp; MQTT/TLS using PSK cipher suite</td></tr>
</tbody></table>

The [next post](/2018/lp-wifi-esp-comparison) compares the low-power performance of the esp32 with
the esp8266.
