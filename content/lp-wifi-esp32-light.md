---
name: lp-wifi-esp32-light
title: ESP32 Maintaining an Association
date: 2018-12-02
thumbnail: "/img/low-power-wifi/esp32-deep-sleep-secure-mode-3-all-sq.png"
categories:
- low-power-wifi
- low-power
- wifi
- esp32
---

The esp32 can also maintain an association with an access point while going to sleep on
its own when it is idle. Is this a good alternative to periodically waking up from deep-sleep?<!--more-->

There are actually several different modes that allow the esp32 to maintain an association and
reduce power when it is not active. The modes are described in
[the API reference](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/power_management.html)
and they all take advantage of the Wifi power save mode and DTIM
interval (see my [earlier post](/lp-wifi-association) on the topic).
To save power, the device tells the access point (AP) that it's turning its radio off
and the AP holds packets destined to the device until the device wakes-up again.
In addition, the AP includes a flag in its
beacons to tell the device whether it has packets pending so the device only neeeds to turn RX on
periodically and check tothe beacon.

### Modem sleep

The arduino framework automatically supports the "modem sleep" mode where the esp32 turns off
the radio but keeps the cpu cores running.
I tested this by modifying [the MQTT test app](https://github.com/tve/low-power-wifi/)
so it connects to Wifi and the MQTT server in the
`setup` function and then publishes to a loop-back topic once every few seconds in the `loop`
function.

I ended up using an open access point and unencrypted MQTT connection so I could analyze the packets
more easily. (I did later test using a secure AP and MQTT over TLS and the observed behavior is almost
indistinguishable.)
A tcpdump running on the MQTT server showed the following for a loop iteration:

```
13:25:39.900240 IP 192.168.0.165.52345 > 192.168.0.14.1883: Flags [P.], seq 24:48, ack 25, win 4976, length 24
13:25:39.900471 IP 192.168.0.14.1883 > 192.168.0.165.52345: Flags [P.], seq 25:49, ack 48, win 29200, length 24

13:25:40.133252 IP 192.168.0.165.52345 > 192.168.0.14.1883: Flags [.], ack 49, win 4952, length 0
```

That's pretty impressive: the first packet contains the publish request (using MQTT QoS 0), the
server turns around and sends the message back on the pre-existing subscription 230 _microseconds_
later and piggy-backs the TCP ACK on the same packet. Finally the esp32 ACKs the server's packet at
the TCP level. This is as minimal as one could wish for!

The following scope capture shows the loop iteration and power consumption:

![MQTT pub-sub with modem sleep](/img/low-power-wifi/esp32-modem-sleep-mqtt.png)

The yellow trace is high from the start to the end of the `loop` function.
That takes under 15ms and includes publishing a message and then waiting for it to come back on a
subscription to the same topic.

The blue trace shows the power consumption at a resolution of 50mA per vertical division. One can
easily make out the modem sleep power level around 40mA, the radio RX power level around 115mA,
and the short transmit spikes. The RX power spikes every 100ms are also clearly visible where the
esp32 captures the AP's beacon to see whether it has a packet pending.

The shape of the RX-active period should look familiar from the equivalent
[esp8266 scope captures](/lp-wifi-esp8266-assoc): the esp32 powers the radio up, transmits
a "exiting power-save mode" packet, then transmits and receives data packets, and finally lingers for some
time to give the AP an opportunity to send additional packets. Eventually the esp32 transmits a
"entering power-save mode" packet and turns the radio off.

In the esp8266 the time spent lingering was around 80ms, in the esp32 this got shortened to 50ms,
which is better in the many cases where there are no packets. I wish it was smarter and cut that
time down more. For example, it could determine whether the AP is idle by monitoring the radio and,
when that is the case, conclude early on that the AP must not have any queued packet.

The odd feature of this scope capture is the fact that there are two wake-up periods with activity
spaced only 150ms apart when the `loop` function iterations are several seconds apart. Given that
neither of the two wake-up periods start at a beacon interval the reason must be other than the
AP having some packet pending.
The answer about what is happening comes from the tcpdump above: the TCP stack on
the esp32 delays the final ACK by about 100ms, i.e., longer than the lingering time. Delaying ACKs
is a common TCP optimization that reduces the number of packets transmitted. It works well on the
server end where delaying the ACK allowed it to be piggy-backed on the response data packet but it
back-fires in the case of the ACK from the esp32 and doubles the power-cost of the whole exchange.

Unfortunately it is not easily possible to disable the ACK delay. It is possible to disable
the Nagle algorithm by calling `WiFiClient.setNoDelay` but that does not affect delayed ACKs.
Looking at the LwIP code, a TCP_QUICKACK socket option is not implemented and in fact
there is no configuration setting to affect ACK delay: the code to send the ack when receiving
data on a connection is
[here](https://github.com/espressif/esp-lwip/blob/046fadde072b5fca94bea84c16cce5ecbfd6948e/src/core/tcp_in.c#L1541)
and would need to be changed to `tcp_ack_now(pcb)`.

Overall all this is interesting to understand the timing behavior of the esp32 but from a power
point of view the minimum consumption of 40mA is not what I'm looking for.

### Dynamic Frequency Scaling with Light Sleep

In order to reduce the power consumption further the esp32 needs to go into light sleep mode when it
is idle. In the ESP IDF framework this is done by enabling power management (a compile-time option)
and then calling a few functions as described in
[the documentation](https://docs.espressif.com/projects/esp-idf/en/latest/api-reference/system/power_management.html).
Unfortunately the arduino framework does not currently support enabling the power management because
its interaction with peripheral clocks is non-trivial.

To evaluate the potential of the frequency scaling and light sleep I started with the `power_save`
ESP IDF example app. At first I had difficulties getting it to work, which resulted in a 
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
for 50ms and sends a power-save-on packet before dropping to sleep.

The capture also shows some timer activity every 100ms and the light-sleep power save level. Not
really discernible in this capture is the fact that the light sleep consumption is about 1.1mA.

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

The next post ... stay tuned...

[Low-power Wifi series index](/categories/low-power-wifi)
