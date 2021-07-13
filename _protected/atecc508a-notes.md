---
layout: encrypted
title: Private - Notes on ATECC508A
tags: arduino atecc508a  
category: prototyping 
description: private notes on the datasheet of ATECC508A...
comments: true
author: Andrew M. Zhang
---

 Abstract: I was trying to learn a bit more about this chip and its features. The Sparkfun lib and cryptolibauth (for python) seem like good starting points, but I wanted to try the more obscures stuff. I will post explanations ot the features below. All of this data can be gleemed from the publically available datasheet linked here and mirror.

# Organization

Being a security device that stores keys, this device has EEPROM. The EEPROM is split into several zones

## Data Zone

This zone is 1,208 bytes and split into 16 read/write memorly *slots* that can be used to store keys, sigs, certs, etc. The access policy of each slot is determined by its respective config settings in the config zone. These policies only kick in if the datazone and OTP lock (LockValue - probably should have been named ValueLock...) is set. More on this later.

## Configuration Zone

This zone is split into various sized chunks that determine some global settings, such as I2C addr, device id, etc. There are also sixteen 2byte regions that correspond to the data slots. These are the individual configs of the dataslots. This zone can be modified until the configuration lock (LockConfig - again shitty name... names should be nouns not verbs...) is set. At which point the configuration zone is stuck. 

The policies dictated in this zone do not kick in until after the Value Lock (LockValue) is set, as mentioned above in Sec. Data Zone). The reason for this will be discussed in the lock section. 

# Locks

 This chip has several locks. The configzone lock. The datazone and OTP lock. And individual locks per slot.

 ## Configuration Zone Lock

 Once this lock is set. The configuration zone cannot be written to. It can still be read, however. The reason why this is separate from the DataZone + OTP Lock (LockValue) is so that a manufacturer can write the config, lock it, send it to a subcontractor who writes additional data to the data zones (perhaps their own private keys), locks it, and then sells the product. 

 ## 


























