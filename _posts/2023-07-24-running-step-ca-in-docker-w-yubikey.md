---
title: Running step-ca in docker w/ Yubikey
tags: AndyWebServices Certificate Authority Yubikey smallstep step-ca 
category: tech
description: How to run step-ca in docker while storing private keys on Yubikey 
giscus_comments: true
author: Andrew M. Zhang

---

***Note: I don't have an extra yubikey+rpi, so I can't really verify these instructions and am relying on memory***

#### Abstract
Carl Tashian of Smallstep wrote a blog post regarding [Building a Tiny Certificate Authority For Your Homelab](https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/). 
This guide provides very complete instructions on how to install a step-ca certificate authority on a Raspberry Pi 4 
and how to store the private keys securely on a Yubikey.

The only problem is I like using Docker and I generally dislike running stuff on bare metal. Unfortunately, there are ß≤
not any clear instructions on how to use the `smallstep/step-ca:hsm` image with a Raspberry Pi.

I recommend trying to set up what's in the guide, so you have a general idea of how the file structure works. I'm going
to document approximately how I setup step-ca.

Read the following to understand the parts of a PKI https://smallstep.com/blog/everything-pki/

The key issue with running this on a docker is that there's a bug in step-ca's dockerfile.hsm. See the final step
for the fix

# Requirements
* Yubikey
* USB stick

# Setup Root and Intermediate keys onto USB stick

The goal is to have the root and intermediate keys on a USB stick. There are 2 ways to go about this.

1. You can follow the Carl Tashian guide linked above until "Part 3: Configuring your CA". I recommend adding a luks
partition so the keys aren't just sitting on a usb stick unencrypted. This guide contains a few simple commands to
create a luks USB stick.https://github.com/drduh/YubiKey-Guide#backup
2. You can also invoke something like this to generate the keys on the usb stick
```bash
# modified from `Manual Installation` here https://hub.docker.com/r/smallstep/step-ca

docker pull smallstep/step-ca:hsm
docker run -it -v /mnt/secret_mount/:/home/step smallstep/step-ca:hsm ca init --remote-management 
# Setup name constraints now if you're going to use them.
# /mnt/secret_mount is where the luks partition is mounted
# Follow the instructions that print to CLI
```

# Load the keys onto the yubikey

Check the instructions in the Carl Tashian guide for how to import the keys onto the yubikey. 

# Remove pcscd from the bare metal machine

Uninstall the yubikey stuff after this step. The `pcscd` service will lock up the yubikey to the host meaning we can't pass it to
the docker container.

# Create a skeleton setup without secret keys

Run the following. Answer the prompts / set CLI variables the same way you answered them before
```bash
# modified from `Manual Installation` here https://hub.docker.com/r/smallstep/step-ca
docker pull smallstep/step-ca:hsm
docker run -it -v /home/pi/docker/mnt/step/:/home/step smallstep/step-ca:hsm ca init --remote-management 
```

Go into `/home/pi/docker/mnt/step` and delete the contents of the `secrets` folder. Copy certs from the mounted usb
to the `certs` folder. Copy config from the mounted usb to the `configs` folder.

Edit the file in `config/ca.json` to read something like. Edit values where appropriate
```json
{
        "root": "/home/pi/docker/mnt/step/certs/root_ca.crt",
        "federatedRoots": [],
        "crt": "/home/pi/docker/mnt/step/certs/intermediate_ca.crt",
        "key": "yubikey:slot-id=9c",
        "kms": {
            "type": "yubikey",
            "pin": "123456"
        },
        "address": ":443",
[...]
```

Edit defaults.json to use the fingerprints of our original certs (ie the first time we ran `step ca init`). 

A this point we're fully setup to run the cert auth

# Random stuff

Run step 4 from https://hub.docker.com/r/smallstep/step-ca because the container expects a password file. You can make the
file empty. Not a big deal

# Start step-ca and pass Yubikey

****I'm hoping that this section becomes fixed in the future. Track the PR here: <insert PR here>****

Clone the repo `github.com/smallstep/certificates.git`. Copy docker/Dockerfile.hsm to repo root.

So there's a bug as of the writing of this guide 07/25/23. The issue is that the user `step` in the container doesn't
have adequate permission to run pcscd service. Thus you're going to need to include these 3 lines in the Dockerfile
```dockerfile
# Change the first line to use golang:bullseye, ie
FROM golang:bullseye AS builder  # If we're building on the rpi, this ensures we have the right glibc version

# Insert this after `RUN chown step:step /run/pcscd` but before `USER step`
RUN groupadd pcscd          # Create the pcscd group
RUN usermod -aG pcscd step  # Adds the user step to pcscd group
```

Then build this image and launch
```
# From the ../certificates directory we cloned earlier
docker build --no-cache -t some-catchy-name:hsm . # build docker image

# You can figure out the bus and usb id via `lsusb` command. This does tend to change when you unplug and replug. You can
# maybe use udevadm to set up a symlink. See the Carl Tashian guide for that setup. I think passing the --privileged 
# flag will let the contianer access all devices
docker run -d -p 443:9000 -v /home/pi/docker/mnt/step:/home/step --device /dev/bus/usb/001/008:/dev/bus/usb/001/008 some-catchy-name:hsm
```

There's probably a better way to deal with the device passing. I'll figure it out later. 


