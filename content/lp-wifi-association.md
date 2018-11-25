---
name: lp-wifi-association
title: Low-Power Wifi using Power Save Mode
date: 2018-11-24
thumbnail: "/img/low-power-wifi/legacy_power.JPG"
categories:
- low-power-wifi
- low-power
- wifi
---

Instead of disconnecting from the AP before going to sleep and then reassociating it is also
possible to maintain the association while going to sleep.<!--more-->

This is made possible at the Wifi level by the Power Save Mode (PSM) which was included in the
original 802.11 standard. PSM leverages the periodic beacon sent out by access points and addresses
the problem of sending packets from the access point (AP) to low-power devices without requiring that the
device keep its receiver on all the time.
(Sending _to_ the access point is easy in that the device can TX at will.)

In a nutshell, the AP does not transmit packets to low-power devices as soon as it receives them.
Instead, it buffers them and includes a flag for the specific device in its beacons.
The device then arranges timers so it wakes up just in time to turn RX on and receive the
beacon. If its flag is set, indicating that it has one or more packets pending at the AP, 
the device sends a poll packet to the AP requesting delivery and the AP transmits the packet.

The device doesn't actually have to wake-up for each and every beacon, instead it can send an
interval to the AP telling it that it will check only every Nth beacon. Often N=3 is used,
which would be a wake-up every 300ms assuming a standard 100ms beacon interval.
See https://alethea.in/legacy-power-save-mode/ for a more detailed explanation.

There is a newer very similar mode called WMM Power Save which operates according to the same
principle but instead of including a map in beacons the low-power device wakes up periodically at
whatever rate it chooses and sends a probe to the access point to prompt delivery of any buffered
packets. The upside is that the device can fully control the interval instead of being bound by the AP's
beacon interval. The downside is that each wake-up requires a transmission instead of the
lower-consumption RX, although the TX duration is shorter than the RX duration necessary to ensure
reception of the beacon given clock rate uncertainty.

There are additional modes, such as scheduled WMM Power Save and Power Save Multi-poll but they get
more complicated and I haven't seen small devices using them.

How well power save mode works can unfortunately only be answered by experimentation.
Worse, a fundamental problem here is that the device
is not really in control of its own power consumption,
it really depends more on what is happening on the network than on the device itself.
This means that on the test bench when using an isolated test-Wifi network everything may look
perfect, but then in a real-world deployment the activity on the network can raise the power
consumption by orders of magnitude.

The [next post](/lp-wifi-esp8266-assoc) describes experiments using the Espressif esp8266.

[Low-power Wifi series index](/categories/low-power-wifi)
