---
layout: post
title:  "Connecting Remote Controlled Blinds to Alexa Smart Home"
excerpt: ""
date:   2018-09-08 12:36:00
categories: 
tags: rust alexa microcontrollers
---

My intuition is to begin with a bit about the pragmatic motivations for the project, but if I’m being honest this came from a place of ‘I bet I could do this’ rather than ‘what my life needs isn’t remote controlled blinds, it’s _voice_ controlled blinds’. 

Anyway. My coworking space recently bought some remote controlled blinds. This is a story about how I connected them to Alexa.

The idea is to have an architecture like so

![The architecture](/images/alexa-blinds/arch.png)

And we’d flow data through it 

![Data flow](/images/alexa-blinds/action.png)

Digging into the details, we’ll start with the last step in our diagram: How do we replicate the remote’s commands?

We’re off to an easy start as the remote lists its transmitting frequency on the back. Similar devices will have an FCC id that can be referenced to find the same information.

![SDR](/images/alexa-blinds/remote.jpg)

Next we’ll rely on a software defined radio and the amazing [Universal Radio Hacker](https://github.com/jopohl/urh) to record and analyze the remote’s messages. 

![SDR](/images/alexa-blinds/sdr.jpg)

My SDR, similar to [https://www.aliexpress.com/item/Ham-Radio-Receiver-100KHz-1-7GHz-full-Band-UV-HF-RTL-SDR-USB-Tuner-Receiver-USB/32894229299.html](https://www.aliexpress.com/item/Ham-Radio-Receiver-100KHz-1-7GHz-full-Band-UV-HF-RTL-SDR-USB-Tuner-Receiver-USB/32894229299.html).

After recording a few taps of the remote, we can use Universal Radio Hacker to decode the message

![URH](/images/alexa-blinds/urh-analysis.png)

The process couldn’t have been easier. Looking at the raw signal, I took a (lucky?) guess that it used [ASK modulation](https://en.wikipedia.org/wiki/Amplitude-shift_keying), and the software will then detect the bit length and produce an interpretation of the signal.

Next step is to check that we’re on the right path by transmitting the same code ourselves and making sure we can control the blinds. I used two components for this, an [ESP8266](https://www.aliexpress.com/item/New-Wireless-module-NodeMcu-Lua-WIFI-Internet-of-Things-development-board-based-ESP8266-with-pcb-Antenna/32656775273.html) and a [433Mhz transmitter](https://www.aliexpress.com/item/RF-wireless-receiver-module-transmitter-module-board-Ordinary-super-regeneration-315-433MHZ-DC5V-ASK-OOK-for/32298304710.html)

Connected on a breadboard along with an LED give a cue when it’s transmitting

![The device](/images/alexa-blinds/device.jpg)

Now we can write some quick and dirty code to see whether we can move the blinds

{% gist 260bb5d048b55b7364b39cad19f54f86 %}

And it works! I’m not sure I expected it to.

_I feel self conscious posting that example using a String to store the command. Later we’ll switch to a more efficient encoding, but as a proof of concept, well, it proves the concept._

Moving on. 

Next we’ll look at exposing the microcontroller over an HTTP interface. The plan here is to make a simple HTTP server to receive high-level actions like ‘Raise the blinds’ and pass them on the microcontroller in the form of the bits necessary to take that action.

This means we’ll have a process that’s listening on two ports. One port for incoming HTTP requests, and one port that our microcontrollers will connect to as they come online. We’ll then use the device information from the HTTP request to find the microcontroller to communicate with, look up its action mapping (e.g. for Stu’s blinds controller, the action ‘up’ means send 10101011’ at 300 bits / second), and encode and transmit the action to the microcontroller. To serialize the action we’ll use

`<MESSAGE TYPE: 1 byte><BIT DURATION: 4 bytes><MSG LENGTH: 4 bytes><MSG: MSG LENGTH bytes>`

The microcontroller side of this code was pleasingly simple to write. The ESP8266 has a built in wifi chip, and nice library support for creating a TCP connection. All we need to do is to write code to connect to our server and then handle the two message types our server sends: keep alives and actions. 

We’re close now. We can control the blinds with curl, so we’re already living large. The last step is connecting our HTTP interface to the Alexa Smart Device system. 

A couple of things need to happen here, and I’ll only touch on them as they’re well covered in [Amazon’s docs](https://developer.amazon.com/docs/smarthome/steps-to-build-a-smart-home-skill.html). We need to write a lambda function that exposes our [device’s interface](https://developer.amazon.com/docs/device-apis/list-of-interfaces.html) and responds to voice commands from Alexa, and then provide an [OAuth endpoint](https://developer.amazon.com/docs/smarthome/steps-to-build-a-smart-home-skill.html#provide-account-linking-information). All are well documented on Amazon.
