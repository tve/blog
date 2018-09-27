---
title: SPI kills LoRa radio
date: 2018-07-21
thumbnail: "/img/sports-tracker/20180924_145431.jpg"
categories:
- Sports tracker
- LoRa
---
I spent some time this week troubleshooting the LoRa communication
problems I encountered when testing long range links from the ocean
sports tracker. It led me to the SPI bus...
<!--more-->

![Tracker with dipole comparison](/img/sports-tracker/20180924_145431.jpg# fr w-40pct ml-3)
_Tracker with the small antenna inside and a proper full dipole to its
right for size comparison._

To recap: I'm sending tracking packets from the
tracker to the GW and the GW responds with an ACK to each packet. Both
sides use the same TX power but I see 9-10dB less SNR at the node end. I
thought it had something to do with the GW being a small linux SBC and
having more noise or something, so I put together a simple STM32-based
node to respond with ACKs like the GW does. I also displayed more info:
SNR, RSSI, and noise measurements taken there was no packet in
the air. It was all rather confusing, in part because the SNR on the
sx1276 LoRa chips doesn't report above about 8dB and at close range
while testing it's difficult to actually go that low.

Long story short, I saw that RSSI was within 1dB on both ends but SNR
was off by 10dB. So both ends were getting the same signal strength but
not the same signal-to-noise. Thus noise had to be the issue. But when
I tried to measure the noise floor I got about the same values on both
ends. Mystery. The answer turned out to be that reception on the linux
end is fully interrupt driven while at the STM32 end the uC keeps polling
the radio chip's registers to see the "RX done" flag being raised. That
polling over the SPI bus creates enough noise to reduce sensitivity
by 10dB!

I did a quick test. Since the ACK comes in immediately I inserted a
long-enough sleep before starting to poll such that the first poll happens
after the reception is complete. Tada! This brought SNR to the same level
on both ends! I then tried something different: I changed the slew rate on
the SPI pins to "2Mhz", which has the effect of creating ~6-8ns rise and
fall times instead of ~1.6ns. With the original polling that reduced the
sensitivity loss from the 10dB to about 1dB. So that creates some options:
a) sleep for the expected time required for the ACK to come in, b)
use an interrupt for RX-done, c) reduce the slew rate on the SPI pins.

All this is fine and dandy for the test node I was using to try out
the different combinations, but on the tracker it's somewhat more
complicated. First of all, the SPI bus is used to drive not only the
radio but also the flash chip and the display. That means that options
(a) and (b) are useless unless  the code also avoids talking to these
two devices while an ACK might be coming in. At the slow LoRa data rates
an ACK takes 300-400ms and currently I'm updating the display twice a
second, so there's not much time left.

The other problem is that the GPS and the BLE interface are constantly
sending stuff to the STM32 via serial ports and that is bound to also
create noise. Those lines don't go to the radio module, but still. The
effect of that is unknown.

As a first step forward I ended up reducing the slew rate on all outputs
from the STM32 chip. Testing at medium range seems to indicate a 2dB
loss in sensitivity, which is not great but a start. I yet have to do
a long range test.

On the current STM32L082 chip package I'm out of pins, so I can't separate
the radio's SPI bus from the display's. I will probably have to live with
the 2dB sensitivity loss for my prototype, if that's what it comes out to
be at long distance. I'm pondering what to do for the "real" version. One
option is to use a uC chip with more pins so I can dedicate one SPI bus
just to the radio, under the assumption that it's just the SPI bus that
matters and that I can decouple power sufficiently. The other would be to
break things up and use separate PCBs and uCs for radio vs. display, etc.
