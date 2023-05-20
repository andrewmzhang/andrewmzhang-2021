---
layout: post
title: Custom TLD Over Private Network
tags: AndyWebServices
category: tech
description: Pretending to be a legit enterprise with my own TLD
comments: true
author: Andrew M. Zhang
---

# Intro

Tailscale is a Wireguard based VPN software. I use on my devices so they can access each remotely and securely. I love
how the setup is extremely minimal - simply run Tailscale on a device you want to add to the VPN and Tailscale handles
the rest. One annoyance is that the built-in DNS only resolves the device's hostname without a tld, ie if you add a
machine with hostname `machineA` you can access a webpage hosted on it via `https://machineA` but there's no way to get
it to work with `https://machineA.ctld`. This page documents how I got a private TLD over Tailscale to work.

Also I insist on TLS because I like the green lock symbol. This needs to look legit or else people won't believe that
AndyWebServices El El Sí is a legit company and I won't be able to get investors and a 100 trillion evaluation.

To clarify. These are the requirements:

* Any and only devices connected by my Tailscale network should be able to access the custom tld network
* I need to use a custom tld such as `hostname.ctld`. I'm going to use `.ctld` as the TLD for the purpose of this guide
* HTTPS also needs to work

# Resolving a custom TLD over Tailscale

## Resolving custom TLD

Tailscale handles secure connection between devices connected to the Tailscale network. Each device on the Tailscale (
TS)
network receives a static TS IP (100.xx.yy.zz) and a non-TLD domain. The non-TLD domain is resolved by a local DNS
resolver that comes
with the TS install.

We want to add a custom TLD to this setup. First step is we're going to need a dedicated DNS server to resolve `*.ctld`
domains. I recommend the following setup:

1. Get a raspberry pi
    1. If you're going to copy my network setup, make it run Ubuntu
2. Connect it to the TS network. Let's pretend it has TS IP of `100.100.100.101`
3. Setup `dnsmasq`
4. Manually set the dns entries in `/etc/hosts`
    1. If you're running Ubuntu default install you might need to edit the cloud init template, check the comments in
       your `/etc/hosts`
    2. It should like like this

```bash
# /etc/hosts
# tailscale_IP hostname
myMachineA.ctld 100.100.100.123
```

You only need to do this once. The Tailscale IP is static. These IPs are not accessible if you're not hooked into the
VPN,
so although I recommend you setup dnsmasq into your tailscale network interface, doing it on `0.0.0.0` probably isn't
a security risk

## SplitDNS

Ok so we have a dns server. Now we need to make sure that devices on the TS network actually use it. The best way to do
this
is to go into the Tailscale Admin Console and setup SplitDNS. Make `*.ctld` resolve with the tailscale ip of the dns
server `100.100.100.101`

This way once a device connects to the Tailscale Network, it automatically tries to use the rpi dnsmasq resolver for
only
domains with our custom tld. This way we stay efficient; `google.com` is still resolved with local dns and only `*.ctld`
gets
sent over our TS network's dns server which might be close by.

# TLS Certificates ( The Green Stuff )

In order to look legit and get the green lock in our browsers and use `https` without our browsers freaking out on us,
we're
going to need some TLS certificates. If you're familiar with what TLS is, skip the rest of this subsection. If the
following
explanation is not detailed enough, there's plenty of more in-depth resources linked below.

What is TLS? TLS is part of Private Key Infrastructure (PKI). If you have a client and server and they want to
communicate securely, the client will need to know what the server's public key is. The DNS server tells you the IP of
`machineA.ctld`, but when `machineA.ctld` responds with its public key, how do you know it's legit and was not
corrupted or maliciously edited in transit over the network? I suppose that communications between TS devices are
secure,
but the rest of the internet can't make that assumption and without the server's pubkey we can't communicate securely.
The TLS certificate is a attestation signed by a trusted party that says `This pubkey belongs to machineA.ctld`. The
trusted party is referred to as the Certificate Authority (CA).

There are 3 parts to be aware of. There's the root certificate. This is a self-signed certificate that contains the CA's
public key. This root cert needs to be installed on all machines that want TLS to work on our custom TLD. The private
key
used to sign the root cert is usually stored cold offline, since root private key compromise is kinda unrecoverable.

The intermediate certificate is signed by the root certificate. The private intermediate key is usually kept online,
since
it needs to sign leaf certificates. If an intermediate cert is compromised or expired, we can bring the root key out of
cold storage and issue a revocation.

The leaf cert is the certificate that `machineA.ctld` needs to provide to our browser in order to prove that it's
public key is legit. These can be revoked by the intermediate cert if `machineA.ctld` goes rogue.

TLS certificates can be generated by the CA and scp'd to the server. I use this solution for devices where for some
reason I can't run ACME properly. ACME is an automated method of generating certificate signing requests and proving
to the CA that the requesting server actually owns the domain.

## ACME

If you've ever used LetsEncrypt or certbot, then you're familiar with ACME. ACME is a set of protocols that servers
can use to prove to CAs that they own the domain name that want certs for. The point of a certificate is to say
certify that `machineA.ctld` is the owner of a particular pubkey. How can the ACME protocol verify this fact? There
are 2 methods. The less common method is a DNS certification. The idea is the client server contacts the ACME server and
claims that it owns `machineA.ctld` and provides a Certificate Signature Request (CSR) to be signed. The ACME server
demands that the client server drop a particular string into the DNS record's TXT section. If the server truly does own
`machineA.ctld` and it's authoritative nameserver has an API for setting domain records, then this operation can be
performed by the server and verified by the ACME. The DNS server we're using is dnsmasq and I don't really know how to
do that.

The 2nd method is more popular anyways. The challenge is the ACME server will ask the client server to put up some
particular
HTML content at `machineA.ctld/.well-known/acme-challenge/<TOKEN>`. ACME will then try to retrieve this page and if it's
there, it will sign our CSR. This way is a lot easier and does not depend on which DNS service we happen to be using.

## Step CA

The idea is to follow this guide https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/ here and setup an
ACME server on a raspbery pi. You don't need to load the keys onto a Yubikey, but I think it's good to do security
things properly and I get them for free at work! I opted not to use the infnoise generator dongle. ACME will run on port
443 and the DNS server runs on 53, so they won't collide. Remember to change the domain in the guide
from `tinyca.internal`
to `<your choice>.ctld`. I called mine something like `ctldca.ctld`

After this step, you will be able to request certs for your custom domain.

## Install the root cert

To prevent your browser and servers requesting certificates from freaking out that it doesn't recognize the ctld CA,
remember to install the root certificate to your device's truststore. To test if you did this correctly, you can drop
into a shell and run `curl https://machinea.ctld`. If you get an error 60, it means you messed up and your machine
doesn't recognize the CA.

I recommend putting the root certificate in a cloud drive or something so you can quickly install it onto new devices.

## Requesting a cert

There are a few ways of requesting a certificate from your CA. Check here for
guides: https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/. Personally I think `acme.sh` is the easiest to
use.

### Some pitfalls in cert requesting

I'll probably split this off into separate guides when I get the chance.

#### HAProxy

Certain reverse proxies, such as HAProxy, will want your leaf certificate private key and certificate in 1 file. You
might need to cat them together if using `acme.sh`. A reverse proxy may also hog port 80 and 443 which are needed to
do the ACME challenge. Remember to carve out a path to your ACME client. I have something like this for HAProxy:

```bash
# In /etc/haproxy/haproxy.cfg
frontend public
    bind :::80 v4v6
    # Redirect requests to this path towards the ACME cleint instance on port 8888
    acl letsencrypt-acl path_beg /.well-known/acme-challenge/
    use_backend letsencrypt-backend if letsencrypt-acl
    
    # Set the certificate
    bind :::443 v4v6 ssl crt /home/pi/.acme.sh/octoprint.aws.pem

# This is the acme.sh port
backend letsencrypt-backend
        server letsencrypt 127.0.0.1:8888
```

```
# In crontab
# Request certs, place them in /some/config/dir. Follow the guide for details on configuring this. 
0 * * * * /some/dir/acme.sh --cron --home "/some/config/dir/" --force --httpport 8888 

# Combine the privKey and the cert. Order matters here
1 * * * * cat /some/config/dir/machineA.ctld.key /some/config/dir/machineA.ctld.crt > /some/config/dir/machineA.ctld.pem
```

You also need to reload the certificate cause HAProxy is too dumb to pick up that you edited it. Drop this script into
a file, edit your crontab to run it. Also maybe edit out the octoprint stuff:

```bash
#!/bin/bash
echo “========================== SET SSL CERT ==========================“
echo "$(cat /home/pi/.acme.sh/octoprint.aws.pem)"
echo -e "set ssl cert /home/pi/.acme.sh/octoprint.aws.pem <<\n$(cat /home/pi/.acme.sh/octoprint.aws.pem)\n" | socat tcp-connect:localhost:9999 -
echo “========================== SHOW SSL CERT - before ==========================“
echo "show ssl cert */home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -
echo “========================== COMMIT SSL CERT ==========================“
echo "commit ssl cert /home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -
echo “========================== SHOW SSL CERT - after ==========================“
echo "show ssl cert /home/pi/.acme.sh/octoprint.aws.pem" | socat tcp-connect:localhost:9999 -
```

#### HomeAssistant

This one is a pain in the ass. Haven't gotten this up and running yet. I'll write another post to document this.

