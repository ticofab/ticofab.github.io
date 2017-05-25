---
layout: post
title: First steps with IoT and LoRaWAN
excerpt: "First steps into the IoT world."
modified: 2017-5-25
tags: [iot, lora, LoPy, Sodaq]
comments: true
---

**The First False Step**

Some time ago I had purchased a [LoPy]() module and a [SodaqONE]() module. The LoPy seemed somewhat simpler to get started with, so I unpacked it first and started reading the online documentation.

![The unpacked LoPy](/images/2015-07-29-last-button-clicked.png)

It didn't take me long to realise that the LoPy module is only the gateway to the LoRa netowrk, and that I would need the expansion board to connect it to my computer and update its firware. Doh.

Let's move on to my Sodaq ONE package.

**The Sodaq ONE trials**

Following the handy [getting started guide](http://support.sodaq.com/sodaq-one/sodaq-one/), I installed the Arduino IDE and the additional Board Manager for the Sodaq ONE. Unfortunately the guide stops at the end of the IDE installation, and in order to go further I had to follow the official SODAQ instructional video to be found [here](https://www.youtube.com/watch?v=G3OovUzntz0).
At the end, a [simple 'blink' example](https://github.com/GabrielNotman/AutonomoTesting/blob/master/Pin%20IO/blink/blink.ino) is suggested. Copy/pasting the code won't be enough tho, as the IDE will complain that

{% raw %}
'LED_BUILTIN' was not declared in this scope
{% endraw %}

This happens as the Sodaq ONE board has different defines for its leds. We can use LED_RED, LED_GREEN or LED_BLUE alternatively.

This is all cool, but towards the end the next problem came up: the board disappears shortly after connecting it with the following goodbye message:

{% raw %}
** Boot-up completed successfully!
The USB is going to be disabled now.
{% endraw %}

It seems that this is quite normal (?), and if you upload your sketch before this message appears then it won't disconnect again. I fiddled around a bit with the blink example - mostly to get my fingers to type C-style code again - and this is the results. Everything works as expected.


{% highlight c %}
#define TEST_LED LED_BLUE

void setup() {
  // Wait for SerialUSB or 10 seconds
  while ((!SerialUSB) && (millis() < 10000));
  SerialUSB.println("Serial monitor opened...");
  pinMode(TEST_LED, OUTPUT);
}

void setupLight(int value) {
  digitalWrite(TEST_LED, value);
}

void loop() {
  setupLight(HIGH);
  SerialUSB.println("...");
  delay(500);
  setupLight(LOW);
  SerialUSB.println("---");
  delay(500);
}
{% endhighlight %}

This sketch (and others to follow) is available on a [github repo](https://github.com/ticofab/sodaq-one-sketches).

The next step I wanted to achieve was to connect my device to [The Things Network](https://www.thethingsnetwork.org). I live in Amsterdam, which is where the network was born, so that should be a well-covered area. I even contributed back then with the [TTN Android SDK](https://github.com/ticofab/The-Things-Network-Android-SDK), now deprecated. I followed the instructions as per the [IoT days 2017 workshop](https://github.com/TheThingsNetwork/workshops/blob/master/IoT-Tech-Day.md), and grabbed the Sodaq ONE [Universal Tracker](https://github.com/SodaqMoja/SodaqOne-UniversalTracker) to get things going, but with little success. Even with all settings correct, the device disconnects and I can't see any incoming data in the TTN console.

{% raw }
Settings:
  GPS (OFF=0 / ON=1)         (gps=): 1
  Fix Interval (min)         (fi=): 1
  Alt. Fix Interval (min)    (afi=): 0
  Alt. Fix From (HH)         (affh=): 0
  Alt. Fix From (MM)         (affm=): 0
  Alt. Fix To (HH)           (afth=): 0
  Alt. Fix To (MM)           (aftm=): 0
  GPS Fix Timeout (sec)      (gft=): 120
  Minimum sat count          (sat=): 4
  OTAA Mode (OFF=0 / ON=1)   (otaa=): 0
  Retry conn. (OFF=0 / ON=1) (retry=): 0
  ADR (OFF=0 / ON=1)         (adr=): 1
  ACK (OFF=0 / ON=1)         (ack=): 0
  Spreading Factor           (sf=): 7
  Output Index               (pwr=): 1
  DevAddr / DevEUI           (dev=): <my_dev_addr>
  AppSKey / AppEUI           (app=): <my_app_id>
  NWSKey / AppKey            (key=): <my_network_key>
  Num Coords to Upload       (num=): 1
  Repeat Count               (rep=): 0
  Status LED (OFF=0 / ON=1)  (led=): 0
  Debug (OFF=0 / ON=1)       (dbg=): 0
Initializing LoRa...
** Boot-up completed successfully!
The USB is going to be disabled now.
{% endraw %}

As a first day into IoT experiments, I had fun. I am curious about the recent Android Things initiative. But first let's get this connection to work! 




