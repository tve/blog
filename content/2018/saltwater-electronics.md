---
title: Do electronics like saltwater?
date: 2018-07-26
thumbnail: "/img/sports-tracker/20180807_205955_sml.jpg"
project: sports-tracker
tags:
---
Picture yourself swimming in the ocean and looking at your electronics
bathing in the saltwater next to you...<!--more-->

Well, that's what happened to me a couple of weeks ago! Surfing down a
wave I tipped to the side falling out of the surfski. I somehow managed
to flip open the latch of the tracker's case with my toes. So there I
was, swimming next to the surfski and seeing the box open and the PCB
plus the battery plus the little foam dividers all happily bathing in
the ocean water! Before the next wave washed everything away I somehow
managed to stuff everything back into the box without too much water
and lock it back up. What's your guess, anything salvagable here?

![LiIon cell](/img/sports-tracker/20180807_205955_sml.jpg# fr w-30pct ml-3)
_Li-Ion cell with gunked top after saltwater bath_

Back home I inspected the damage. The top of the battery had
a whole lot of dark goo. It must be toast. But still registered
3.1V! Interesting... Lots of white crud all over the PCBs. I spent some
time cleaning them with denatured alcohol and then plugged everything
into a current-limited power supply and started to apply power. To my
great suprise the display came on and showed the normal info!?

What didn't seem to just work was the bluetooth heart rate info. After
a good hour of troubleshooting I concluded that the bluetooth radio's
output stage must have gotten fried in the bath. Probably the radio was
transmitting and the output got short-circuited. The module seemed
to work perfectly and respond to commands as expected, but it simply
wasn't seeing any of the bluetooth devices around my desk. I ended up
ordering a new one and when I put it in it worked right off the bat.

Looks like I got really lucky that not more broke and that everything
was held in place by the cables long enough for me to collect it before
it would sink! For my next prototype I will make sure that the latch can't
be accidentally opened with the toes!!!

To be continued...
