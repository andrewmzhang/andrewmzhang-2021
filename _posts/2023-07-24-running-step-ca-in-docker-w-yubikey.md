---
layout: post
title: Running step-ca in docker w/ Yubikey
tags: [AndyWebServices, Certificate, Authority, Yubikey, smallstep, step-ca] 
category: tech
description: How to run step-ca in docker while storing private keys on Yubikey 
giscus_comments: true
author: Andrew M. Zhang

---

#### Abstract

Carl Tashian of Smallstep wrote a blog post regarding [Building a Tiny Certificate Authority For Your Homelab](https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/). 
This guide provides very complete instructions on how to install a step-ca certificate authority on a Raspberry Pi 4 
and how to store the private keys securely on a Yubikey. This post is written assuming familiarity with this guide.
Namely, be familiar with the process of running the `step ca init` twice, once to setup the private keys off-host (ie usb)
, then again to setup the directory structure. Certs must then be copied from the off-host device to the directory 
structure. Private keys on the off-host device are loaded into a Yubikey. 

The only problem is I like using Docker and I generally dislike running stuff on bare metal. Unfortunately, there are
not any clear instructions on how to use the `smallstep/step-ca:hsm` image with a Raspberry Pi with a Yubikey. My guide 
will roughly follow the same steps except using docker.

***NOTE*** There is currently a bug in step-ca's `dockerfile.hsm`. See the final step of the guide for a quick fix.

Read the following to understand the parts of a PKI https://smallstep.com/blog/everything-pki/

## Requirements
* Rpi
* Yubikey
* USB stick

## Setup Rpi

I tried this using both Ubuntu and Raspbian on Rpi. I think it'll work on any Rpi OS that supports Docker. Do whatever
you need to do to install Docker.

## Setup Root and Intermediate keys onto USB stick

The goal is to have the root and intermediate keys on a USB stick, just like in Carl's guide. Though, I recommend
using a LUKS encrypted USB stick to hold the keys. You can steal the commands for setting that up here: https://github.com/drduh/YubiKey-Guide#backup

For the remainder of the guide I'll assume your usb, encrypted or otherwise, is mounted here `/mnt/usb/`

```bash
# modified from `Manual Installation` here https://hub.docker.com/r/smallstep/step-ca

docker pull smallstep/step-ca:hsm

# Fill out the prompts with your own answers
docker run -it -v /home/pi/docker/data/step:/home/step andywebservices/step-ca:hsm step ca init --name="AndyWebServices CA"\
 --dns="awsca.aws,172.19.0.100" --address=":443" --provisioner="admin@andywebservices.com" --deployment-type standalone \
 --remote-management --acme --admin-name admin@andywebservices.com
there is no ca.json config file; please run step ca init, or provide config parameters via DOCKER_STEPCA_INIT_ vars
✔ Deployment Type: Standalone                                                   # Select Standalone
What would you like to name your new PKI?
✔ (e.g. Smallstep): TinyCA                                                      # Give it a name. This will appear in yoru certs
What DNS names or IP addresses will clients use to reach your CA?               
✔ (e.g. ca.example.com[,10.1.2.3,etc.]): ca.internal,192.168.0.101,10.10.10.10  # You can list a bunch of these. Values can be adjusted layer in /mnt/usb/config/ca.json
What IP and port will your new CA bind to? (:443 will bind to 0.0.0.0:443)      
✔ (e.g. :443 or 127.0.0.1:443): :443                                            # You probably want :443
✔ (e.g. you@smallstep.com): email@example.com                                   # Your email
Choose a password for your CA keys and first provisioner.
✔ [leave empty and we'll generate one]:
✔ Password: my.password.is.unique                                               # Pick a sensible password

Generating root certificate... done!
Generating intermediate certificate... done!

✔ Root certificate: /home/step/certs/root_ca.crt
✔ Root private key: /home/step/secrets/root_ca_key
✔ Root fingerprint: 0d134a66060ff7540f3cd304c81d354ded26d4b57d4d33f5b445ad555b7c0335
✔ Intermediate certificate: /home/step/certs/intermediate_ca.crt
✔ Intermediate private key: /home/step/secrets/intermediate_ca_key
badger 2023/07/26 06:04:17 INFO: All 0 tables opened in 0s
✔ Database folder: /home/step/db
✔ Default configuration: /home/step/config/defaults.json
✔ Certificate Authority configuration: /home/step/config/ca.json
✔ Admin provisioner: email@example.com (JWK)
✔ Super admin subject: step

Your PKI is ready to go. To generate certificates for individual services see 'step help ca'.

FEEDBACK 😍 🍻
  The step utility is not instrumented for usage statistics. It does not phone
  home. But your feedback is extremely valuable. Any information you can provide
  regarding how you’re using `step` helps. Please send us a sentence or two,
  good or bad at feedback@smallstep.com or join GitHub Discussions
  https://github.com/smallstep/certificates/discussions and our Discord
  https://u.step.sm/discord.
```

### Optional: Set up name contraints

Follow the instructions here to setup up name constraints if you're going to set it up for the root cert. This is probably
good practice. If you don't use name constraints and your CA is hijacked, then it can issue certs for any domains (ie google.com). 
Usually name constraints are set on the intermediate cert but for custom TLDs and internal stuff, constraining the
root cert is acceptable and is the only way to show clients that you're not going to sign for stuff you don't own (ie google.com).

I use an internal TLD. You should set the permitted domains to also include the one you use for emails or else I think
you could get issues. Ie I setup my name constraints to sign `.customtld`, `mydomain.com` (I own this), and `.mydomain.com` 

## Load the keys onto the yubikey

Check the instructions in the Carl Tashian guide for how to import the keys onto the yubikey. You might need to install
`yubikey-manager` (ie ykman) and `pcscd`

https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/#import-the-ca-into-the-yubikey

## Remove pcscd from the bare metal machine. Other cleanup

Uninstall the yubikey stuff after this step. The `pcscd` service on the host machine will lock up the yubikey to the 
host meaning we can't pass it to the docker container.

Unmount the encrypted drive. You don't need it anymore

## Create a skeleton setup without secret keys

I'm going to assume that the docker mount for your step-ca server will be under `/home/pi/docker/mnt/step`

Run the following. Answer the prompts / set CLI variables the same way you answered them before
```bash
# modified from `Manual Installation` here https://hub.docker.com/r/smallstep/step-ca
docker pull smallstep/step-ca:hsm
docker run -it -v /home/pi/docker/mnt/step/:/home/step smallstep/step-ca:hsm step ca init --remote-management
# Follow the prompt the same way you filled out before

# Delete the secret keys from the skeleton. Copy the configs and certs. Do NOT copy the secret keys. These are on the Yubikey
rm -rf /home/pi/docker/mnt/secrets/*
cp /mnt/usb/config/* /home/pi/docker/mnt/step/config/
cp /mnt/usb/certs/* /home/pi/docker/mnt/step/certs/
```

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
        "address": ":443"
// [...]
```

At this point, we're fully setup to run the cert auth

## Random stuff

You need a password file. It's just a thing the docker entrypoint for `step-ca` expects.

```bash
touch /home/pi/docker/mnt/step/secrets/password
```

## Start step-ca and pass Yubikey

***I'm hoping that this section becomes fixed in the future. Track the PR here: <insert PR here>***

Clone the repo `github.com/smallstep/certificates.git`. Copy `docker/Dockerfile.hsm` to repo root.

So there's a bug as of the writing of this guide 07/25/23. The issue is that the user `step` in the container doesn't
have adequate permission to run pcscd service. Thus you're going to need to include these 3 lines in the Dockerfile
```dockerfile
# Change the first line to use golang:bullseye. Otherwise it might not build right on your RPI due to GLibC versioning issues
# If we're building on the rpi, this ensures we have the right glibc version.
# You may need to adjust as raspbian moves to newer versions
FROM golang:bullseye AS builder  

# Insert this after `RUN chown step:step /run/pcscd` but before `USER step`
RUN groupadd pcscd          # Create the pcscd group
RUN usermod -aG pcscd step  # Adds the user step to pcscd group
```

Then build this image and launch
```
# Execute the following from repo root
docker build --no-cache -t some-catchy-name:hsm . # build docker image

# You can figure out the bus and usb id via `lsusb` command. This does tend to change when you unplug and replug. You can
# maybe use udevadm to set up a symlink. See the Carl Tashian guide for that setup. I think passing the --privileged 
# flag will let the contianer access all devices
docker run -d -p 443:443 -v /home/pi/docker/mnt/step:/home/step --device /dev/bus/usb/001/008:/dev/bus/usb/001/008 some-catchy-name:hsm
```

## Messing with the udev rules

The above *should* work, but if it doesn't, try running the container as root user with the option `-u 0`

I got mine to work with the `step` user (in the container) by setting these udev rules in the host. I'm not a udev expert
but I think these changes let `pcscd` work correctly in the container without the `-u 0` option.
```
# IN file /etc/udev/rules.d/75-yubikey.rules
ACTION=="add", SUBSYSTEM=="usb", ENV{PRODUCT}=="1050/406/*", TAG+="systemd", SYMLINK+="yubikey", GROUP="pcscd", MODE="0666", RUN+="/bin/chgrp pcscd $root/$parent"
ACTION=="remove", SUBSYSTEM=="usb", ENV{PRODUCT}=="1050/406/*", TAG+="systemd"
```

## My image

You can get a copy of my rpi step-ca image via `docker pull andywebservices/step-ca:hsm`. 


