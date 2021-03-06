---
name: lp-wifi-association
title: Low-Power Wifi using Power Save Mode
date: 2018-11-24
thumbnail: "/img/low-power-wifi/legacy_power.JPG"
project: low-power-wifi
tags:
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

The device doesn't actually have to wake-up for each and every beacon, instead the Ap can be
configured to include the packet-pending flags only in every Nth beacon.
Often N=3 is used, which would be a wake-up every 300ms assuming a standard 100ms beacon interval.

There is a newer very similar mode called WMM Power Save which operates according to the same
principle but instead of including a map in beacons the low-power device wakes up periodically at
whatever rate it chooses and sends a probe to the access point to prompt delivery of any buffered
packets. The upside is that the device can fully control the interval instead of being bound by the AP's
beacon interval. The downside is that each wake-up requires a transmission instead of the
lower-consumption RX, although the TX duration is shorter than the RX duration necessary to ensure
reception of the beacon.

There are additional modes, such as scheduled WMM Power Save and Power Save Multi-poll but they get
more complicated and I haven't seen embedded devices using them.

A fundamental problem with Power Save Mode is that unrelated activity on the network can cause
packets to be queued for delivery to the sleeping device, causing a wake-up.
This means that the device is not really in control of its own power consumption.
A simple example of such activity are the ARP requests seen in
the deep-sleep mode: suppose there are 20 esp8266's on the network, each of which wakes up every 20
seconds. That would result in an ARP packet being broadcast every second on average as some esp8266
wakes up and tries to find its server. The result would be that an unrelated esp8266 in Power Save
Mode would get woken up once per second!

All this means that how well power save mode works can only be answered by experimentation.
On the test bench, when using an isolated test-Wifi network, everything may look
perfect, but then in a real-world deployment the activity on the network can raise the power
consumption by orders of magnitude.

The [next post](/2018/lp-wifi-esp8266-assoc) describes experiments using the Espressif esp8266.
