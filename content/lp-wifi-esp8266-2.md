---
name: lp-wifi-esp8266-2
title: ESP8266 with Optimized Periodic Wakeup
date: 2018-11-23
thumbnail: "/img/low-power-wifi/ap-test-sq.png"
categories:
- low-power-wifi
- low-power
- wifi
- esp8266
---

This post dives into optimizing deep-sleep with periodic wakeup on the esp8266 and ends
with some calculations about expected run-time on battery.
<!--more-->

If you haven't read the [previous post](/lp-wifi-esp8266-1) on the unoptimized version you probably
should go back so you understand my set-up.

The following scope capture shows the wake cycle after optimization.
The Wifi probes scanning all channels and the DHCP queries have disappeared.
Note the horizontal scale of 50ms per division. That means that everything happens in about
175 milliseconds! I also changed from `ESP.deepSleep()` to `ESP.deepSleepImmediate()` to avoid the
lingering at the end of the activity before cutting the power.

![Wake-up cycle](/img/low-power-wifi/ap-test-mode-3-all-annot.png# w-100pct)
_Scope capture of a full wake cycle passing all possible parameters to `WiFi.begin` and `WiFi.connect`._

Now that the horizontal scale is a bit bigger we can see that the
microcontroller initialization takes about 80-90ms. The Wifi
sequence then starts with a probe request on the correct channel followed by an immediate response
by the AP:

```
0.788s -> Probe Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 24.0 48.0 Mbit]
0.793s <- Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] CH: 6
0.795s -> Acknowledgment RA:74:da:38:06:52:32
0.820s <- Beacon (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
0.851s <- Beacon (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
```

After a 114ms idle period (the beacons in the above packet dump don't matter) comes
the association with the AP which starts with a de-association in order to clear any old
state in the AP:

```
0.907s -> DeAuthentication (5c:cf:7f:05:c0:db): Deauthenticated because sending station is leaving (or has left) IBSS or ESS
0.907s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.907s -> Authentication (Open System)-1: Successful
0.907s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.912s <- Authentication (Open System)-2:
0.912s -> Acknowledgment RA:74:da:38:06:52:32
0.913s -> Assoc Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 24.0 48.0 Mbit]
0.913s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.916s <- Assoc Response AID(1) :: Successful
0.917s -> Acknowledgment RA:74:da:38:06:52:32
```

Now that the esp8266 has an association it confirms that no-one else has its IP address using a
gratuitous ARP which is immediately followed by an ARP request to get the MAC address of the server
for the TCP connection. Note that this second ARP request is repeated after 5ms because evidently
the server didn't answer.

```
0.919s -> ARP, Request who-has 192.168.0.106 tell 192.168.0.106, length 28
0.920s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.922s <- Beacon (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
0.925s -> ARP, Request who-has 192.168.0.106 tell 192.168.0.106, length 28
0.930s -> ARP, Request who-has 192.168.0.2 tell 192.168.0.106, length 28
0.930s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.935s -> ARP, Request who-has 192.168.0.2 tell 192.168.0.106, length 28
0.936s <- ARP, Reply 192.168.0.2 is-at b8:97:5a:90:54:5a, length 46
0.937s -> Acknowledgment RA:74:da:38:06:52:32
```

With MAC address in hand the esp8266 opens the TCP connection and should transfer the data but what
the capture shows is this:

```
0.938s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [S], seq 6509, win 2144, options [mss 536,nop,nop,sackOK], length 0
0.939s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.940s -> Acknowledgment RA:74:da:38:06:52:32
0.942s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [R.], seq 1, ack 1, win 24584, length 0
0.943s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.943s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [S], seq 6509, win 2144, options [mss 536,nop,nop,sackOK], length 0
0.944s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.948s -> Acknowledgment RA:74:da:38:06:52:32
0.949s -> Acknowledgment RA:74:da:38:06:52:32
0.950s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [.], ack 50581235, win 2144, length 0
0.951s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.951s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [P.], seq 4294967212:0, ack 50581235, win 2144, length 84
0.953s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.953s <- Beacon (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] ESS CH: 6
0.963s -> Acknowledgment RA:74:da:38:06:52:32
0.965s -> IP 192.168.0.106.55779 > 192.168.0.2.12345: Flags [F.], seq 0, ack 50581235, win 2144, length 0
0.966s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.975s <- IP 192.168.0.2.12345 > 192.168.0.106.55779: Flags [F.], seq 50581235, ack 1, win 29200, length 0
0.983s <- IP 192.168.0.2.12345 > 192.168.0.106.55779: Flags [F.], seq 50581235, ack 1, win 29200, length 0
0.983s <- IP 192.168.0.2.12345 > 192.168.0.106.55779: Flags [F.], seq 50581235, ack 1, win 29200, length 0
0.992s <- IP 192.168.0.2.12345 > 192.168.0.106.55779: Flags [F.], seq 50581235, ack 1, win 29200, length 0
[a few dozen more of the same...]
```

It starts out with a TCP SYN ("Flags [S]") as expected and the back-to-back Acknowledgment packets
indicate that the monitoring adapter missed some packets but the odd part is the second TCP packet,
which is a connection reset ("Flags [R]"). That's not expected. And the third appears to be a new
SYN for a fresh connection.

Another oddity is that at the end the server (192.168.0.2) sends a pile of FIN packets ("Flags
[F]"), which indicates that the esp8266 has gone to sleep before the connection closed properly
and the server is trying to get an ACK for its FIN.
So basically the server is left hanging trying to close the connection.

I then looked at a longer capture that shows multiple wake cycles and noticed that the esp8266
uses the same source port (55779) each time! Now we can explain the connection reset: the server
can't close the connection at the end of a wake-up cycle and the esp8266 then comes along at the next
cycle with the same &lt;src IP, src port, dst IP, dst port&gt; combination. To the server this
looks like a continuation of the TCP connection, which results in a protocol violation which the
esp8266 eventually resolves using a TCP reset.

Fortunately we can fix this little problem by setting the local port number before opening the
connection using `WiFi.setLocalPortStart()` and then incrementing and remembering it in the RTC RAM.
The result is a clean TCP connection:

```
0.851s -> IP 192.168.0.106.10008 > 192.168.0.2.12345: Flags [S], seq 6509, win 2144, options [mss 536,nop,nop,sackOK], length 0
0.851s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.856s <- IP 192.168.0.2.12345 > 192.168.0.106.10008: Flags [S.], seq 1790078284, ack 6510, win 29200, options [mss
0.op,sackOK], length 0
0.856s -> Acknowledgment RA:74:da:38:06:52:32
0.858s -> IP 192.168.0.106.10008 > 192.168.0.2.12345: Flags [.], ack 1, win 2144, length 0
0.859s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.859s -> IP 192.168.0.106.10008 > 192.168.0.2.12345: Flags [P.], seq 1:85, ack 1, win 2144, length 84
0.861s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.865s <- IP 192.168.0.2.12345 > 192.168.0.106.10008: Flags [.], ack 85, win 29200, length 0
0.865s -> Acknowledgment RA:74:da:38:06:52:32
0.867s -> IP 192.168.0.106.10008 > 192.168.0.2.12345: Flags [F.], seq 85, ack 1, win 2144, length 0
0.868s <- Acknowledgment RA:5c:cf:7f:05:c0:db
0.869s <- IP 192.168.0.2.12345 > 192.168.0.106.10008: Flags [F.], seq 1, ack 86, win 29200, length 0
0.869s -> Acknowledgment RA:74:da:38:06:52:32
```

This connection uses local port 10008, the next wake up will use 10009 and so forth. In this
connection dump we see the expected SYN, SYN-ACK, ACK, 84-byte data packet, ACK, FIN, ACK-FIN.
Missing is a final ACK and I still see FIN retransmissions from the server later on, but at least
they don't interfere with the next connection anymore.

For comparison, the following capture shows the full activity period zoomed-in. (It doesn't match
the previous packet dump exactly, I didn't capture the same iteration this time.)

![Wake-up cycle zoomed](/img/low-power-wifi/ap-test-mode-3-end2-annot.png)
_Magnification of the activity portion of the wake cycle after fixing the local port numbers._

The vertical cursors show that the entire activity period lasts only 31 milliseconds
and the vertical cursors show that the power draw is just around 70mA most of the time.
The total wake cycle still lasts 170-180ms primarily due to the wait-time after the probe requests.

### Secure access points

All the experiments so far have used an open unencrypted access point. This made the packet dump
analysis easy because all is in the clear. But it's also insecure: everything is in the clear and
can be hijacked at will. The question this raises is what happens when using a secure access point?
The following scope capture shows the same sketch when using the same AP with a WPA2 encrypted SSID:

![Wake-up cycle with secure AP](/img/low-power-wifi/ap-secret-mode-3-all-annot.png# w-100pct)
_Scope capture of a full wake-up cycle using a secure WPA2 access point._

The first thing to notice is the horizontal scale: back to 500ms per division!
Thus the wake-cycle went from under 200ms to about 2 seconds, ouch.
There is actually quite some variability with most wake cycles
taking from 2 to 3 seconds. To understand what is happening one has to look at the packet dump,
which starts with probing and associating as usual:

```
  123  7.269s Probe Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 24.0 48.0 Mbit]
  124  7.274s Probe Response (test) [1.0* 2.0* 5.5* 11.0* 6.0 9.0 12.0 18.0 Mbit] CH: 6, PRIVACY
  125  7.276s Acknowledgment
  146  8.271s DeAuthentication: Deauthenticated because sending statione.avior has left) IBSS or ESS
  147  8.271s Acknowledgment
  148  8.271s Authentication (Open System)-1: Successful
  149  8.272s Acknowledgment
  150  8.276s Authentication (Open System)-2:
  151  8.277s Acknowledgment
  152  8.277s Assoc Request (test) [5.5* 11.0* 1.0* 2.0* 6.0 12.0 24.0 48.0 Mbit]
  153  8.277s Acknowledgment
  154  8.280s Assoc Response AID(1) : PRIVACY : Successful
  155  8.282s Acknowledgment
```

While this looks the same as previously the timestamps are not the same. Specifically, there is a
one second pause between the probe response and the authentication.
Looking carefully at the scope trace one can see that the power
consumption is slightly higher during that time compared to most of the other wake time
(except for TX spikes).
I have the strong suspicion that this is the time it takes to perform the crypto necessary for the
subsequent association request, but I can't prove that.

The next set of packets are all encrypted which makes it difficult to make out what they are,
but looking at a tcpdump on the server end is helpful:
```
09.337388 ARP, Request who-has 192.168.0.2 tell 192.168.0.106, length 46
09.337436 ARP, Reply 192.168.0.2 is-at b8:97:5a:90:54:5a, length 28
10.298977 ARP, Request who-has 192.168.0.2 tell 192.168.0.106, length 46
10.299012 ARP, Reply 192.168.0.2 is-at b8:97:5a:90:54:5a, length 28
10.303207 IP 192.168.0.106.10577 > 192.168.0.2.12345: Flags [S], seq 6509, win 2144, options [mss 536,nop,nop,sackOK], length 0
10.303259 IP 192.168.0.2.12345 > 192.168.0.106.10577: Flags [S.], seq 3659718700, ack 6510, win 29200, options [mss 1460,nop,nop,sackOK], length 0
10.307668 IP 192.168.0.106.10577 > 192.168.0.2.12345: Flags [.], ack 1, win 2144, length 0
10.307732 IP 192.168.0.106.10577 > 192.168.0.2.12345: Flags [P.], seq 1:87, ack 1, win 2144, length 86
10.307749 IP 192.168.0.2.12345 > 192.168.0.106.10577: Flags [.], ack 87, win 29200, length 0
10.311969 IP 192.168.0.106.10577 > 192.168.0.2.12345: Flags [F.], seq 87, ack 1, win 2144, length 0
10.312051 IP 192.168.0.2.12345 > 192.168.0.106.10577: Flags [F.], seq 1, ack 88, win 29200, length 0
10.316318 IP 192.168.0.106.10577 > 192.168.0.2.12345: Flags [.], ack 2, win 2143, length 0
```
What this shows is that there are two rounds of ARP resolutions for the server's MAC address
that are spaced 1 second apart (192.168.0.2 is the server). So it appears that the first ARP
response gets lost and the first retry comes after one second. This pattern is so
systematic that it looks more like a bug than a packet loss issue.

While I can't explain all the delays when using a secure access point it is clear that it takes an
order of magnitude longer than when using an open one.
I have not seen a single wake-cycle take less than 2 seconds, also on my home Wifi which I tried
as a sanity check.

### Power calculations

Now that we know the duration of the wake cycle we can construct a simple run-time table assuming
that:

- the power consumption while active is 70mA
- the power consumption while sleeping is 10uA (the bare module consumes less, this provides some
  margin for a regulator and some sensors)
- the wake cycle time is 200ms for an open access point and 2 seconds for a secure one
- a 1000mAh LiPo battery is used to calculate the run-time

sleep [min] | open [mA] | open [days] | open [act%] | secure [mA] | secure [days] | secure [act%]
---:|---:|---:|---:|---:|---:|---:
60 | 0.01 | 3000.2 | 28% | 0.05 | 852.7 | 80%
10 | 0.03 | 1250.4 | 70% | 0.24 | 171.8 | 96%
1 | 0.24 | 171.8 | 96% | 2.27 | 18.4 | 100%
0.33 | 0.71 | 58.7 | 99% | 6.43 | 6.5 | 100%

Notes:

- The columns labeled _open_ and _secure_ refer to an open access point vs. a secure one.
- The columns labeled _[act%]_ show the percentage of power spent during the wake cycle.
- The first row corresponds to a system that wakes up every hour, the last one every 20 seconds.

The conclusion one can draw here is that it is should certainly be possible to run an esp8266 in
this periodic wake-mode for a week to over a year on a single 1000mAh LiPo battery!

The [next post](/lp-wifi-association) takes a brief look at an alternative, which is to
maintain an association with the access point and leverage power save mode to reduce
power consumption.

[Low-power Wifi series index](/categories/low-power-wifi)
