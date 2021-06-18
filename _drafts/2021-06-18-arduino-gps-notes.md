---
layout: post
title: Notes on Arduino GPS Library
tags: GPS SIM808 DFRobot arduino arduino-sim808 
category: tech
---

#### Abstract

I bought DFRobot SIM 808 shield. Cheap shield. However, the library provided by DFRobot is not great. Here are my notes and possible solutions and a crappy fork. 


On Terminology:
* Board refers to the arduino. I used an Arduino Uno Rev3. 
* Shield refers to the DFRobot Shield.

# DFRobot SIM808 Library and Resources

1. [DFRobot SIM808 Github Repo](https://github.com/DFRobot/DFRobot_SIM808)
2. [DFRobot SIM808 Wiki Docs](https://wiki.dfrobot.com/SIM808_GPS_GPRS_GSM_Shield_SKU__TEL0097)


## Pros

The documentation has tool called a Serial Debugger. Super useful. You can issue direct AT (which I understand is some kind of protocol sent over Serial) commands to the shield. There's a typo in the GPS Orientation command. It should say `AT + CGNS PWR = 1`.

## Cons

There are a few issues with this shield. The first issue is that the module communicates with the arduino board through the default TXRX serial ports; pins 0 and 1 are the Arduino Uno. This is a problem because those pins are also connected to the USB Serial line which means:

1. You cannot upload sketches without powering down the shield. 
2. All communication sent to the shield will also show up in USB Serial Monitor.
  1. Any prints done on arduino will show up on the USB Serial Monitor and will be sent to the shield. 
  2. This means debugging is nuts; say you want to print out what the Sim808 library buffer is receiving from the shield and print it w/ Serial.println. This will cause a feedback loop since the library buffer will catch it again. This makes the shield a pain to debug. 

The Library is also not particularly well written. A lot of features are missing, such as trying to get the number of satellites in view. And the design of the library is questionable; IIRC the getGPRMS code read 1 char from the serial stream, adds it toa global buffer, checks to see if the global buffer has accumulated to a valid GPRMS (it ignores all other NMEA streams types), and fails if it doesn't. This means you *need* to wrap calls to getGPS in a while loop since getGPS will fail a lot... 

NOTE: I might need to double check the above. I kinda forgot how the lib actually works and I don't wanna check at 2:26 AM... I will verify/fix this section ASAP...


# blemasle arduino-sim808 library

1. [blemasle library](https://github.com/blemasle/arduino-sim808)

This library appears to be written for the Adafruit FONA? Not sure. 

## Pros

This library has super good logic, also really clean code, fully featured. Some bugs/idosyncracies but I'm not using the right board so eh...  

## Cons

It doesn't work on the DFRobot shield very well. Also no comments so its a pain to figure out how it works. 

# Fixes


## Hardware

The solution here is to unstack the shield and then just use male-to-male jumpers to connect over the power pins. Pins 13 and 12, and connect pins 0,1 on the shield to 2,3 on the board respectively. Then we will use software serial to communicate with the shield, and use the default 0, 1 pins to print Serial messages as per usual. This prevents the Serial prints and board buffer printouts to get feedbacked or whatever. 

NOTE: You still need to manually fiddle with the boot switch. I'm investingating this now.


## Software

We use the Software Serial to 2,3 to communicate with the shield. 

I'm not sure when RDY gets sent. When I had the shield serial connected to board serial default (0,1), I never got the RDY response from the shield and setting the baud rate into non-volatile memory (AT?? command) never held across reboot. You only get the RDY if the baud is set according to the docs. Thus in the default library with the default serial comms, the sim808 gets stuck on init waiting on RDY

Another issue is that if the GPS fix fails (not enough satellites to trilaterate), you cannot get any data from NMEA streams that are available; current time, number of GPS in view, etc. I have fixed this. 



