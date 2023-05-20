---
layout: post
title: Custom TLD Over Private Network
tags: AndyWebServices
category: tech
description: Pretending to be a legit enterprise with my own TLD
giscus_comments: true
author: Andrew M. Zhang
---

# Intro

Tailscale is a Wireguard-based VPN software that I rely on for secure and remote access across my devices. I appreciate
its remarkably minimal setup process - simply installing Tailscale on a device grants it access to the VPN, with the
software taking care of the rest. However, one inconvenience I encountered was the built-in DNS's limitation of
resolving device hostnames without a TLD. For instance, if I add a machine with the hostname machineA, I can access
webpages hosted on it via https://machineA, but unfortunately, Tailscale DNS does not support setting the domain to
something like https://machineA.ctld. This page documents my solution to implementing a private TLD over a Tailscale
network.

I am committed to using TLS because I value the green lock symbol and the assurance it provides. It is crucial for my
setup to appear legitimate; otherwise, people may doubt the credibility of AndyWebServices El El Sí as a legitimate
company. This credibility is essential for attracting investors and achieving a valuation of 100 trillion.

To clarify. These are the requirements:

* Any and only devices connected by my Tailscale network should be able to access the custom tld network
* I intend to use a custom TLD format, such as hostname.ctld. For the purpose of this guide, I will use .ctld as the
  TLD.
* HTTPS functionality

# Resolving a custom TLD over Tailscale

## Resolving custom TLD

Tailscale facilitates secure connections between devices within the Tailscale network. Each device on the Tailscale
network is assigned a static Tailscale IP (100.xx.yy.zz) along with a non-TLD domain. The resolution of this non-TLD
domain is handled by a local DNS resolver that is included with the Tailscale installation.

To incorporate a custom TLD into this setup, the first step is to set up a dedicated DNS server to resolve `*.ctld`
domains. I recommend following the setup outlined below:

1. Obtain a Raspberry Pi device.
    1. If you intend to replicate my network setup, install and configure Ubuntu on the Raspberry Pi.
2. Connect the Raspberry Pi to the Tailscale (TS) network. For the sake of this guide, let's assume it is assigned the
   TS IP address `100.100.100.101`.
3. Install and set up `dnsmasq` on the Raspberry Pi.
4. Edit the `/etc/hosts` file on the Raspberry Pi to manually define the DNS entries.
    1. In case you are using the default Ubuntu installation, you may need to modify the cloud-init template. Please
       refer to the comments in your `/etc/hosts` file for guidance.
    2. The entries in the `/etc/hosts` file should resemble the following format:

```bash
# /etc/hosts
# tailscale_IP hostname
myMachineA.ctld 100.100.100.123
```

You only need to perform this setup once since the Tailscale IP is static. Keep in mind that these IPs are only
accessible when connected to the VPN. While it is advisable to configure `dnsmasq` on your Tailscale network interface,
setting it up on `0.0.0.0` is unlikely to pose a security risk.

## SplitDNS

We must ensure the proper utilization of the DNS server by devices connected to the Tailscale network. To achieve
this, we will proceed with configuring SplitDNS within the Tailscale Admin Console. Our primary objective is to
establish the resolution of `*.ctld` to the Tailscale IP address assigned to the DNS server,
specifically `100.100.100.101`.

By implementing this approach, any device that joins the Tailscale Network will seamlessly attempt to utilize the DNS
resolver of the Raspberry Pi (rpi) running dnsmasq, but solely for domains associated with our custom top-level domain (
`ctld`). This strategy ensures optimal efficiency, as domain resolutions for standard domains like `google.com` continue
to
be handled by the local DNS, while requests for `*.ctld` domains are routed through the Tailscale network's DNS server,
which may incur a higher latency if our rpi is not nearby.

# TLS Certificates ( The Green Stuff )

To enable the use of HTTPS without encountering any browser warnings, obtaining TLS certificates is essential. If you
are already familiar with TLS and its intricacies, you may skip the remaining portion of this subsection. However, if
you require a more detailed explanation, there are numerous comprehensive resources available online.

TLS, which stands for Transport Layer Security, is a crucial component of Private Key Infrastructure (PKI)
framework. When a client and server aim to establish a secure communication channel, the client must be aware of the
server's public key. While the DNS server provides the IP address of `machineA.ctld`, it does not verify the legitimacy
of the server's public key. TLS ensures that the key has not been tampered with or maliciously
altered during transit over the network. While communications between Tailscale (TS) devices are considered secure, the
broader internet cannot make such assumptions. Therefore, without the server's public key, secure communication becomes
impossible. This is where TLS certificates come into play.

A TLS certificate is an attestation signed by a trusted entity known as a Certificate Authority (CA). It asserts that a
specific public key belongs to `machineA.ctld`. Understanding the three levels of TLS certificates is
important. First, the root certificate is a self-signed certificate that contains the public key of the CA. This root
certificate must be installed on all machines requiring TLS functionality for our custom TLD. The private key used to
sign the root certificate is typically kept offline in cold storage, as recovery from a compromised root key is
impossible,
eg you can't sign a revocation if your private key cannot be trusted.

The intermediate certificate is signed by the root certificate and is responsible for signing leaf certificates. Unlike
the root certificate, the private key of the intermediate certificate is usually kept online since it is required to
sign leaf certificates. If an intermediate certificate becomes compromised or expires, the root private key can be
brought out of cold storage to issue a revocation.

The leaf or end certificate, which is presented by `machineA.ctld`, serves as proof that its public key is legitimate.
The
intermediate certificate has the authority to revoke leaf certificates if `machineA.ctld` exhibits malicious behavior.

TLS certificates can be generated by the CA and securely transferred to the server using methods like scp. This approach
is particularly useful for devices where running ACME (Automated Certificate Management Environment) properly is not
feasible. ACME provides an automated mechanism for generating certificate signing requests and validating the domain
ownership to the CA.

By understanding these concepts, we can effectively utilize TLS certificates to establish secure and trusted
communication channels, ensuring the green lock symbol and seamless HTTPS functionality in our browsers.

## ACME

If you have previous experience with Let's Encrypt or certbot, then you are likely familiar with the ACME (Automated
Certificate Management Environment) protocol. ACME is a collection of protocols that servers can employ to demonstrate
their ownership of domain names to Certificate Authorities (CAs) when requesting certificates. The primary purpose of a
certificate is to certify that machineA.ctld is indeed the owner of a specific public key.

ACME offers two verification methods, with the less commonly used method being DNS certification. In this approach, the
client server contacts the ACME server and claims ownership of machineA.ctld. The client server provides a
Certificate Signing Request (CSR) to the ACME server for signing. The ACME server then requests the client server to add
a specific string to the TXT section of the DNS record. If the server genuinely owns machineA.ctld and its authoritative
nameserver supports an API for managing domain records, the server can perform the required operation and have it
verified by the ACME server. However, as we are utilizing dnsmasq as our DNS server, the specific steps for performing
this operation are not within my expertise.

It is worth noting that the DNS certification method is less common compared to the alternative method, which involves
proving ownership through HTTP-based challenges. This method requires the server to respond to a challenge by placing a
designated file at a specified location on the web server. The ACME server then attempts to access the file to validate
ownership. This approach is generally more straightforward to implement and is not dependent on what API your DNS
provider happens to supply.

## Step CA

To proceed with setting up an ACME server on a Raspberry Pi, we can follow the guide provided
at https://smallstep.com/blog/build-a-tiny-ca-with-raspberry-pi-yubikey/. Although the guide incorporates using a
YubiKey for loading keys, it is not necessary for our purposes. However, adhering to proper security practices is always
recommended, and if you have access to YubiKeys, it can enhance the overall security of the setup. I get them for free
at work. There is an option to use the infnoise generator dongle, but we will opt because it costs too much and I don't
get them for free at work. I recommend you use the same Raspberry Pi you're hosting the ctld DNS off of. Note that the
ACME server will run on port 443, while the DNS server will operate on port 53, ensuring that there won't be any
conflicts between the two services.

During the setup process, remember to modify the domain from `tinyca.internal` to a domain of your choice with
the `.ctld` extension, such as `ctldca.ctld`.

Once you have completed this step, you will be able to request TLS certificates for your custom domain, allowing you to
establish secure connections using HTTPS.

## Install the root cert

To ensure that your browser and servers recognize the custom TLD Certificate Authority (CA) and avoid any issues with
certificate verification, install the root certificate into the truststore of your devices. This step
will establish trust in the certificates issued by the custom CA.

To verify if the installation was successful, you can open a shell or command prompt and execute the
command `curl https://machinea.ctld`. If you encounter an error 60, it indicates that the CA is not recognized by your
machine, indicating a problem with the root certificate installation.

For convenience and ease of use, it is recommended to store the root certificate in a cloud drive or a similar storage
solution. This way, you can readily access and install it on new devices whenever necessary.

## Requesting a cert

You can refer to the guides provided at https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/ for detailed
instructions on setting up various ACME clients. However, if you're looking for a straightforward and user-friendly option, I
recommend using acme.sh.

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

