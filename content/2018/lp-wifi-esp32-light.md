---
name: lp-wifi-esp32-light
title: ESP32 Maintaining an Association
date: 2018-12-03
thumbnail: "/img/low-power-wifi/esp32-modem-sleep-mqtt-sq.png"
project: low-power-wifi
tags:
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
interval (see my [earlier post](/2018/lp-wifi-association) on the topic).
To save power, the device tells the access point (AP) that it's turning its radio off
and the AP holds packets destined for the device until it wakes-up again.
In addition, the AP includes a flag in its
beacons to tell the device whether it has packets pending so the device only neeeds to turn RX on
periodically and check the beacon.

### Modem sleep

The arduino framework automatically supports the "modem sleep" mode where the esp32 turns off
the radio but keeps the cpu cores running.
I tested this by modifying [the MQTT test
app](https://github.com/tve/low-power-wifi/esp32-light-sleep-mqtt)
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

Just 3 packets, that's pretty impressive! The first packet contains the publish request (using MQTT QoS 0), the
server turns around and sends the message back on the pre-existing subscription 230 _microseconds_
later and piggy-backs the TCP ACK on the same packet. Finally the esp32 ACKs the server's packet at
the TCP level. This is as minimal as one could wish for!

The following scope capture shows the loop iteration and power consumption:

![MQTT pub-sub with modem sleep](/img/low-power-wifi/esp32-modem-sleep-mqtt.png)

The yellow trace is high from the start to the end of the `loop` function.
That takes under 15ms and includes publishing a message and then waiting for it to come back on a
subscription to the same topic.
Rather impressive, to be honest!

The blue trace shows the power consumption at a resolution of 50mA per vertical division. One can
easily make out the modem sleep power level around 40mA, the radio RX power level around 115mA,
and the short transmit spikes. The RX power spikes to 1125mA every 100ms are clearly visible where the
esp32 captures the AP's beacon to see whether it has a packet pending.

The shape of the active period around the loop iteration should look familiar from the equivalent
[esp8266 scope captures](/2018/lp-wifi-esp8266-assoc#wake-cycle-activity):
the esp32 powers the radio up, transmits
a "exiting power-save mode" packet, then transmits and receives data packets, and finally lingers for some
time to give the AP an opportunity to send additional packets. Eventually the esp32 transmits a
"entering power-save mode" packet and turns the radio off.

In the esp8266 the time spent lingering was around 80ms, in the esp32 this got shortened to 50ms,
which is better in the many cases where there are no packets. I wish it was smarter and cut that
time down more. For example, it could determine whether the AP is idle by monitoring the radio and,
when that is the case, conclude early on that the AP must not have any queued packet.

The odd feature of this scope capture is the fact that there are two long high-power periods
spaced only 150ms apart.
The `loop` function iterations are several seconds apart, so that's not the cause.
The second of the two periods in this scope capture does not start at a beacon interval thus it
must be caused by something other than the AP having some packet pending.
The answer is seen in the tcpdump above: the TCP stack on
the esp32 delays the final ACK by about 100ms, i.e., longer than the lingering time and this causes
a second wake-up period. Delaying ACKs
is a common TCP optimization that reduces the number of packets transmitted. It works well on the
server end where delaying the ACK allowed it to be piggy-backed on the response data packet but it
back-fires in the case of the ACK from the esp32 and doubles the power-cost of the whole exchange.

Unfortunately it is not easily possible to disable the ACK delay. It is possible to disable
the TCP Nagle algorithm by calling `WiFiClient.setNoDelay` but that does not affect delayed ACKs.
Looking at the LwIP code, a TCP_QUICKACK socket option is not implemented and in fact
there is no configuration setting to affect ACK delay: the code to send the ack when receiving
data on a connection is
[here](https://github.com/espressif/esp-lwip/blob/046fadde072b5fca94bea84c16cce5ecbfd6948e/src/core/tcp_in.c#L1541)
and would need to be changed to `tcp_ack_now(pcb)`.

Overall all this is interesting to understand the timing behavior of the esp32 but from a power
point of view the minimum consumption of 40mA at modem-sleep level is not what I'm looking for.
The [next post](/2018/lp-wifi-esp32-dfs) turns on automatic power saving with dynamic frequency scaling.

