# Bluewind

In a recent surprising turn of events I got to connect with some listeners of Jupiter Broadcasting shows, one of which happens to be involved in organising the Berlin Hack and Tell meetup, which is a cool monthly event where people do short demos of cool projects they had hacked together. After some chatting with this funky group of tech enthusiasts I was encouraged to present some of my project that seem to be a perfect fit for Berlin Hack and Tell, so ahead of submitting my lightning talk I wanted to make a blog post here first to put down in words how this project came to be and what it actually is, in case anyone fancies finding out more about it.

## Give yourself no excuse

When it comes to building positive habits, including trying to exercise regularly, I find it important to lower the barrier to entry for myself. One of the ways that I have achieved this is having an indoor cycling setup with a bike on a turbo in front of a TV screen. The bike is always ready, the turbo can be turned on remotely from my phone and controlled by a sports app that suggests me the optimal workout for each day. I try to always have fresh cycling clothes ready. No need for my brain to perform mental gymnastics and participation in a self indulgent competition of making up excuses related to all those minor inconveniances, seemingly rendering me incapable of just jumping on a bike and sweating out some of my frustrations and calories that I could really do with getting rid of.

As a techie the gadgetry side of sport science is very appealing to me and naturally I have eventually paired my turbo trainer with an expensive bluetooth fan which promissed seamless integration into my indoor cycling ecosystem of gadgets. Alas, it was not meant to be. The fan is able to simulate wind speed of up to 25km/h. However the indoor trainer does not simulate correct speed in ERG mode, and that is the mode I predominantly use for my training (using TrainerRoad). The fan can also track your heard rate, but for whatever reason it would never reach the full speed no matter how high my HR had gotten during the workouts I had tried this feature on. This has left me stuck with the manual speed mode, where I must adjust the speed myself between the 4 predefined speed presets. That would not be a big issue had it not been for the fact that in order to use this option I have to either:
- Jump off the bike in order to reach the buttons of this, so called, "smart" fan
- Switch apps on my phone multiple times mid ride from TrainerRoad to the Wahoo app and back in order to adjust

This was not ideal and definitely did not match my usual "low barier to entry" vission for how I like to enjoy my indoor sweating time. And don't get me wrong, this fan is fantastic (pun intended) and does a great job in the core area of its functionality - being a fan and cooling me down, but I decided to take the matter in my own hands and take it up a notch from good to great!

## Research

First things first, we need to see what is already available. A search and a look around had uncovered a github repository which we can use as a code donnor for our project: [https://github.com/garanj/wearwind](https://github.com/garanj/wearwind). Although it is a Java project, an applicataion for Android wear, we can learn a few interesting facts from it. To start, it is controlling the fan by sending it byte arrays of data in the following format:
```java
private val POWER_ON = byteArrayOf(4, 4, 1)
private val POWER_OFF = byteArrayOf(2, 0)
```
[As seen here!](https://github.com/garanj/wearwind/blob/9b7115b4c070d2f2cfa10cb792bb68d4e517a073/app/src/main/java/com/garan/wearwind/FanControlService.kt#L392)

This was a good starting point and following some experimentation I have decoded a few more commands which I found useful, notably sending the unit to sleep:
```python
SLEEP = [0x4, 0x1]
```
And setting a speed of choice:
```python
# Second byte is the speed, values can range from 0x1 to 0x64
MIN_SPEED = [0x2, 0x1]
FULL_SPEED = [0x2, 0x64]
```

[Complete list of commands I use can be seen in the bluewind repo.](https://github.com/octopusx/bluewind/blob/main/headwind/__init__.py)

The beginning of my journey with python and BLE started with doing research on existing libraries and implementations of the BLE stack, where I have discovered the following resources:
- https://github.com/pybluez/pybluez
- https://github.com/ukBaz/BLE_GATT

These seemed like they are used in various projects on github, and the pybluez even comes with handy examples for how to do BLE scans:
- https://github.com/pybluez/pybluez/blob/master/examples/ble/scan.py

Trying to use this library however I quickly learned an interesting fact: BLE is not supported on many laptop or desktop wifi/ble modules! Having had trouble running this code, on a whim I decided to try it our on a raspberry pi 4 I had laying around, and that worked without a hitch. I was able to scan for BLE devices nearby and read their advertised lists of characteristics:
```
service000a = "00001801-0000-1000-8000-00805f9b34fb"                    # Generic Attribute Profile
service000a_char000b = "00002a05-0000-1000-8000-00805f9b34fb"           # Service Changed
service000a_char000b_desc000d = "00002902-0000-1000-8000-00805f9b34fb"  # Client Characteristic Configuration
service000e = "0000180a-0000-1000-8000-00805f9b34fb"                    # Device Information
service000e_char000f = "00002a29-0000-1000-8000-00805f9b34fb"           # Manufacturer Name String
service000e_char0011 = "00002a25-0000-1000-8000-00805f9b34fb"           # Serial Number String
service000e_char0013 = "00002a27-0000-1000-8000-00805f9b34fb"           # Hardware Revision String
service000e_char0015 = "00002a26-0000-1000-8000-00805f9b34fb"           # Firmware Revision String
service0017 = "a026ee01-0a7d-4ab3-97fa-f1500f9feb8b"                    # Vendor specific
service0017_char0018 = "a026e002-0a7d-4ab3-97fa-f1500f9feb8b"           # Vendor specific
service0017_char0018_desc001a = "00002902-0000-1000-8000-00805f9b34fb"  # Client Characteristic Configuration
service0017_char001b = "a026e004-0a7d-4ab3-97fa-f1500f9feb8b"           # Vendor specific
service0017_char001b_desc001d = "00002902-0000-1000-8000-00805f9b34fb"  # Client Characteristic Configuration
service001e = "a026ee0c-0a7d-4ab3-97fa-f1500f9feb8b"                    # Vendor specific
service001e_char001f = "a026e038-0a7d-4ab3-97fa-f1500f9feb8b"           # Vendor specific
service001e_char001f_desc0021 = "00002902-0000-1000-8000-00805f9b34fb"  # Client Characteristic Configuration
```

Characteristics are like fields, or ports, that you can either read from or write to, and after some more experimentation I have identified that in order to control the fan I need to send data to characteristic `a026e038-0a7d-4ab3-97fa-f1500f9feb8b`.

Interestingly however, I had struggled to use the basic pybluez library for this type of communication and have switched over to using an easier to use, more abastract library, i.e. `bleak`: [https://github.com/hbldh/bleak](https://github.com/hbldh/bleak).

It is during this transition that I realised that the BLE protocol is only used for scanning for devices in case of my Headwind fan and actually, once I have found out the characteristic and MAC address to send data to, I no longer rely on the BLE stack for anything and could actually go back to sending data to the fan directly from my laptop. I did not need to scp my code to my local trusty Pi4 no longer from here on out.

## PoC

## End Goal
