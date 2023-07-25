---
title: Running step-ca in docker w/ Yubikey
tags: AndyWebServices Certificate Authority Yubikey smallstep step-ca 
category: tech
description: How to run step-ca in docker while storing private keys on Yubikey 
giscus_comments: true
author: Andrew M. Zhang

---

#### Abstract
Carl Tashian of Smallstep wrote a blog post regarding [Building a Tiny Certificate Authority For Your Homelab](https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/). 
This guide provides very complete instructions on how to install a step-ca certificate authority on a Raspberry Pi 4 
and how to store the private keys securely on a Yubikey.

The only problem is I like using Docker and I generally dislike running stuff on bare metal. Unfortunately, there are ß≤
not any clear instructions on how to use the `smallstep/step-ca:hsm` image with a Raspberry Pi.

I recommend trying to set up what's in the guide, so you have a general idea of how the file structure works. I'm going
to document approximately how I setup step-ca

# Requirements
* Yubikey
* USB stick

# Setup a luks partition on the usbstick

Steal instructions from here. It's not that hard to setup. https://github.com/drduh/YubiKey-Guide#backup

# Create root key and intermediate key

Read the following to understand the parts of a PKI https://smallstep.com/blog/everything-pki/

Run something like this (modified from `Manual Installation` here https://hub.docker.com/r/smallstep/step-ca)

```bash
docker pull smallstep/step-ca:hsm
docker run -it -v /mnt/secret_mount/:/home/step smallstep/step-ca:hsm ca init --remote-management 
# Setup name constraints now if you're going to use them.
# /mnt/secret_mount is where the luks partition is mounted
# Follow the instructions
```

After this, you should have private keys under `/mnt/usb_mount/secrets/*`. Follow the step-ca guide to import these
keys onto the yubikey. 

Uninstall the yubikey stuff after this step. The `pcscd` service will lock up the yubikey meaning we can't pass it to
the docker container

Create a fresh setup under a non-usb mounted location
```bash
# Use the same settings as above
docker run -it -v /home/pi/docker/mnt/:/home/step smallstep/step-ca:hsm ca init --remote-management 

# Copy over the certs and configs we intend to use
cp /mnt/secret_mount/certs/* /home/pi/docker/mnt/certs/
cp /mnt/secret_mount/configs/* /home/pi/docker/mnt/configs/
rm /home/pi/docker/mnt/secrets/*  # Delete the secrets. We don't need 'em
```

# Start step-ca and pass Yubikey

I'm hoping that this section becomes fixed in the future. Track the PR here: <insert PR here>

So there's a bug as of the writing of this guide 07/25/23. You're going to need to include these 2 lines in the Dockerfile
```dockerfile
RUN groupadd pcscd          # Create the pcscd group
RUN usermod -aG pcscd step  # Adds the user step to pcscd group
```

Then build this image and launch
```
docker run -d -p 443:9000 -v /home/pi/docker/mnt/step:/home/step --device /dev/bus/usb/001/008:/dev/bus/usb/001/008 name-of-built-docker:hsm
```

There's probably a better way to deal with the device passing. I'll figure it out later
