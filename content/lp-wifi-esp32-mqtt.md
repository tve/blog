---
name: lp-wifi-esp32-2
title: ESP32 Deep-Sleep Connecting to MQTT
date: 2018-11-30
thumbnail: "/img/low-power-wifi/mqtt-logo.png"
categories:
- low-power-wifi
- low-power
- wifi
- esp32
- mqtt
---

Periodically waking up from deep sleep and sending a short TCP message is interesting, but how does
a real scenario involving sending to and receving from an MQTT server look like?
<!--more-->

The previous posts cover the startup-time to connect to Wifi so this is really an extension of those
tests. To make it fully real, the connection to the MQTT server doesn't just use bare TCP connection 
but additionally encrypts it using TLS. So this is a full-on real scenario. The only aspect that
may not be accurate in some use-cases is that the MQTT server is located on the local network and
not somewhere across the internet. The latter case obviously adds latency and thus will keep the
esp32 awake longer. How much longer depends on the local internet connection, the
distance to the server, and the server load.

![TLS PSK](/img/low-power-wifi/mqtt-psk.png# fr w-30pct)
_Openssl output for TLS PSK connection from esp32_
I had not set-up TLS for my local MQTT server, which runs mosquitto, so I took advantage of that
to skip public key crypto and go straight to the PSK key cipher suites, which are new to me.

The more traditional path using public key crypto is to generate a root certifacte
authority (CA), then generate a cert/private-key combo for the server so the client can auth the
server. And then to either use name/password to auth the client to the server or to generate a
cert/key combo for each client as well. After a while this becomes unwieldy, plus the public
crypto involved is quite heavy, i.e. slow.

### TLS using pre-shared key cipher suites

PSK stands for pre-shared key and means that the client and server share a key. This replaces all
the certs and private keys. 
In addition to the PSK, the client also has a string ID,
and the entire set-up is not more involved than assigning each client an ID and a randomly generated key.
Practically the same as assigning each client a name/password combo, but more powerful and secure.

Using the PSK cipher suites the client authenticates the server because the server has to prove that
it knows the key. Similarly the server authenticates the client using this key.
In this process the key itself is
never transmitted, so attempting to connect to the wrong server does not reveal the key. In the
end the client and server negotiate an encrypted connection just like with the more common
public key cipher suites.

The esp32 framework (ESP-IDF) uses mbed TLS and conveniently it supports the PSK cipher suites, they
just need to be enabled. I prepared a pull-request for the arduino framework and as of Dec 3rd the
PSK cipher suites are supported in the git master version of the framework (easy to install using
platformio's "staging" version of the framework).

### Test application

As a test I use a simple app that connects to Wifi, connects to the mqtt server, subscribes to a
topic, then publishes a short message to that topic and finally waits for the message to come back
on the subscription.
This simulates a use-case where the esp32 periodically wakes up, takes a measurement and sends it to
the MQTT server, but also subscribes to a topic so it can be sent commands to make changes or
perhaps to stay awake and accept interactive commands for a period if time.

The first text output of the app is shown below. The duration labeled "Mqtt" is from start-up to
connecting to the MQTT server, including crypto. The duration labeled "Sleep" then further includes
subscribing, publishing, and waiting for the return message.

```
===== Deep sleep MQTT test starting =====
Wifi connect in 948ms
Mode 3, Init 18 ms, Wifi 948 ms, Mqtt 977 ms, seq=138, SSID test, IDF v3.2-dev-39-gaaf12390
Sleep at 1001 ms
```

A scope capture of this process looks as follows:

![ESP32 deep sleep and MQTT scope capture](/img/low-power-wifi/esp32-deep-sleep-mqtt-annot.png)
_Scope capture of the esp32 waking from deep sleep and doing pub-sub with an MQTT server over TLS._

Here is becomes obvious that Wifi is fast: the added time compared to the simplest TCP connection is
at most 100ms, i.e. less than 10% of the total. So as long as the server is fast at responding it
doesn't really matter what the esp32 does once it has gone through all the trouble of associating
with the access point. (I should caution that using public key crypto may take longer, I didn't try.)

Stay tuned for more posts about the Esp32's Wifi power consumption...

[Low-power Wifi series index](/categories/low-power-wifi)
